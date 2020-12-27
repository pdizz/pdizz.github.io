---
layout: post
title:  "Terraforming Minecraft - Launching a Minimal Minecraft Server"
date:   2020-09-22 0:00:00 -0700
categories: devops terraform aws minecraft
series: minecraft-terraform
order: 2
---

The first, most essential resource for any application in AWS is a VPC, or virtual network that will contain all your application resources within your chosen region.

### Creating a VPC and subnets

While you can define all your terraform resources in the same file, it is usually better to break them up into logical pieces for your own sanity. Create a file in the same directory as your `main.tf` called `vpc.tf` and define your first (default) VPC.

{% highlight shell %}
resource "aws_vpc" "main" {
  cidr_block       = "172.16.0.0/24"
  instance_tenancy = "default"
}
{% endhighlight %}

In addition we also need to define VPC Subnets which are IP blocks for your network that provide public or private IP addresses for the specific Availability Zones in your region. Reference your VPC id `aws_vpc.main.id` in your subnets. We can start with just one public subnet in us-west-2a for now.

{% highlight shell %}
resource "aws_subnet" "usw2a_public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "172.16.0.0/26"
  map_public_ip_on_launch = true
    
  tags = {
    Name = "minecraft"
  }
}
{% endhighlight %}

Run `terraform apply` and review and confirm the plan and Terraform will create the VPC and Subnet in your region.

{% highlight shell %}
Terraform will perform the following actions:

# aws_subnet.usw2a_public will be created
...
# aws_vpc.main will be created
...
Plan: 2 to add, 0 to change, 0 to destroy.
{% endhighlight %}

That is technically all you need to start launching EC2 instances, but before you can actually connect to anything from the internet you need to set up a route to your VPC from the internet. The most basic configuration to accomplish this is with an internet gateway and a single public subnet. 

### Basic VPC Networking Configuration

First define an internet gateway resource attached to your VPC, then a route table with a route for your internet gateway with a CIDR block of `0.0.0.0/0`. Finally create an association resource to attach the route table to your public subnet, which will automatically create the local route to your subnet.

{% highlight shell %}
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "minecraft"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "minecraft"
  }
}

resource "aws_route_table_association" "usw2a_public" {
  subnet_id      = aws_subnet.usw2a_public.id
  route_table_id = aws_route_table.public.id
}
{% endhighlight %}

Run `terraform apply` again to finish setting up your VPC.

{% highlight shell %}
Terraform will perform the following actions:

# aws_internet_gateway.main will be created
...
# aws_route_table.public will be created
...
# aws_route_table_association.usw2a_public will be created
...
Plan: 3 to add, 0 to change, 0 to destroy.
{% endhighlight %}

Now your EC2 instance is accessible from the internet.

### Launching a Minimal EC2 instance and Installing the Minecraft Server

To launch an instance in EC2 first define an aws_instance resource in your chosen availability zone and subnet. You'll also need to choose the `instance_type`, or size of server you want based on how you anticipate using your server.

> For demonstration I went with `t3a.medium` which is about the cheapest instance that can reliably serve a couple of players. Check out the [Minecraft Dedicated Server Requirements](https://minecraft.gamepedia.com/Server/Requirements/Dedicated) for some guidance on what your needs will be and review the available [ec2 instance types](https://aws.amazon.com/ec2/instance-types/) to find one that will work for you.

Finally we need to specify an AMI. You can use any linux flavor you choose like Ubuntu or Amazon Linux. I'm using a CentOS 7 image maintained by centos.org (free subscription from the AWS Marketplace) and these install instructions are for CentOS. In your terraform project root create a file called `ec2.yaml` and define an ec2 instance.

{% highlight shell %}
resource "aws_instance" "minecraft" {
  ami           = "ami-0bc06212a56393ee1" # centos 7 minimal
  instance_type = "t3a.medium"
  key_name      = "minecraft"

  availability_zone = "us-west-2a"
  subnet_id = aws_subnet.usw2a_public.id

  root_block_device {
    delete_on_termination = true
  }
}
{% endhighlight %}

You should also create a security group for the instance, which will allow access on the ports the Minecraft game client needs to connect to your server. In `ec2.yaml` define a security group associated with your VPC. To run a minecraft server, you need to allow incomming connections to TCP and UDP on port 25565 from anywhere. In the security group define `ingress` rules for each requirement. Also define an `egress` rule to allow the ec2 instance to connect to anywhere so it can talk out. You can also include an `ingress` rule to allow SSH connections on port 22 for convenience, but remember this is a public instance.

> Generally it is not a good idea to allow SSH connections from anywhere as it will eventually become a target for attacks. Some better options would be to only allow SSH connections from your IP address, a bastion server within your VPC, or through a VPN, but that is outside the scope of this article.

{% highlight shell %}
resource "aws_security_group" "minecraft" {
  name        = "minecraft"
  description = "minecraft server"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "ssh"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["77.191.142.21/32"]
  }
  
  ingress {
    cidr_blocks      = ["0.0.0.0/0"]
    description      = "ping"
    from_port        = 8
    protocol         = "icmp"
    self             = false
    to_port          = -1
  }

  ingress {
    description = "minecraft game"
    from_port   = 25565
    to_port     = 25565
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "minecraft game UDP"
    from_port   = 25565
    to_port     = 25565
    protocol    = "udp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
{% endhighlight %}

Now associate the security group in your ec2 instance resource.

{% highlight shell %}
resource "aws_instance" "minecraft" {
  ami           = "ami-0bc06212a56393ee1" # centos 7 minimal
  instance_type = "t3a.medium"
  key_name      = "minecraft"

  vpc_security_group_ids = [aws_security_group.minecraft.id]

  availability_zone = "us-west-2a"
  subnet_id = aws_subnet.usw2a_public.id

  root_block_device {
      delete_on_termination = true
  }
}
{% endhighlight %}

Once the instance is running, we could log in and run the commands to install Minecraft and run the server, but it would be better to automate that so we can launch any number of servers at will. One way to accomplish this is providing a script in the instance `user_data` which will run automatically when the instance is launched.

{% highlight shell %}
resource "aws_instance" "minecraft" {
  ami           = "ami-0bc06212a56393ee1" # centos 7 minimal
  instance_type = "t3a.medium"
  key_name      = "minecraft"

  vpc_security_group_ids = [aws_security_group.minecraft.id]

  availability_zone = "us-west-2a"
  subnet_id = aws_subnet.usw2a_public.id

  root_block_device {
    delete_on_termination = true
  }

  user_data = <<-EOF
    #!/bin/bash
    yum -y install java
    useradd --shell /bin/false --home-dir /opt/minecraft minecraft
    yum -y install wget
    wget -qO /opt/minecraft/server.jar https://launcher.mojang.com/v1/objects/f02f4473dbf152c23d7d484952121db0b36698cb/server.jar
    echo "eula=true" > /opt/minecraft/eula.txt
    cd /opt/minecraft && nohup /usr/bin/java -Xms2048M -Xmx2048M -XX:+UseG1GC -jar server.jar nogui &
    EOF
}
{% endhighlight %}

Finally run `terraform apply` to launch your new instance.

{% highlight shell %}
Terraform will perform the following actions:

# aws_instance.minecraft will be created
...
# aws_security_group.minecraft will be created
...
Plan: 2 to add, 0 to change, 0 to destroy.
{% endhighlight %}

Now you should have a working Minecraft server with a public IP address you can connect to from anywhere. You can run `terraform show` to view the state of your infrastructure including the public IP address, or view it in the AWS console. Start a multiplayer game with that IP address and you can play in your new world.

> Is that it? Well there are a couple things we could improve to make our server more user friendly. Having nothing but an IP address for a name doesn't inspire much confidence. Plus whenever you replace your server with a new instance (or even stop and start your instance) your server will have a new IP address and you'll have to update DNS records or notify all your users of the new address. It would be much better to have a custom hostname for our server, and a static IP address to give it some permanance.

### Staking your Claim

To give our Minecraft server a static IP address, we can give it an Elastic IP, which is a static IP address that can be attached to an EC2 instance or other AWS resources. In `ec2.tf` define an `aws_eip` resource to claim a static IP for your server. If you ever need to replace your server with a new instance you (or Terraform) can just attach the Elastic IP to the new instance.

{% highlight shell %}
resource "aws_eip" "minecraft" {
  instance = aws_instance.minecraft.id
  vpc      = true
}
{% endhighlight %}

Finally, if you're already managing your domain in Route53, define a Route53 record to assign a hostname to the Elastic IP to give your server a permanent address.

{% highlight shell %}
resource "aws_route53_record" "pdizz_minecraft" {
  zone_id = "ZJO2WR3LOVAR9D"
  name    = "minecraft.pdizz.com"
  type    = "A"
  ttl     = "300"
  records = [aws_eip.minecraft.public_ip]
}
{% endhighlight %}
