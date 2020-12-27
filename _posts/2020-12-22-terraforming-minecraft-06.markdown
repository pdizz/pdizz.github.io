---
layout: post
title:  "Terraforming Minecraft - Using Modules for Repeatable Infrastructure"
date:   2020-12-22 0:00:00 -0700
categories: devops terraform aws minecraft
series: minecraft-terraform
order: 6
---

In [Preserving and Migrating Data with Volumes]({% post_url 2020-11-23-terraforming-minecraft-05 %}) we briefly touched on previewing infrastructure changes by creating additional testing resources in our terraform config. It would be even better to create an entirely separate testing environment, with it's own VPC and instance, so we can preview changes to any part of our infrastructure without breaking the production environment. We can achieve this more elegantly using Terraform Modules.

### Creating a Terraform Module

Modules in Terraform are very flexible, to the point where the hard part in creating modules is just deciding what to call them and where to draw the lines between parts of your infrastructure. In fact we've already created one module since a module is just a directory of Terraform configs. Our current `terraform/` directory is actually called the *root* module.

> How you structure your modules is up to you. I'm borrowing again from Yevgeniy Brikman's [A Comprehensive Guide to Terraform](https://blog.gruntwork.io/how-to-create-reusable-infrastructure-with-terraform-modules-25526d65f73d) where he recommends separating variables and outputs into their own files like this:

{% highlight text %}
terraform/
|-- modules/
|   `-- ec2/
|       |-- main.tf
|       |-- outputs.tf
|       `-- variables.tf
`-- main.tf
{% endhighlight %}

Let's start by moving our Minecraft EC2 instance and dependent resources into a module. Create the `modules` directory and the `ec2` module with `main.tf`. Now we have to choose which resources belong in the module. I'm going with the instance and security group. I also want to include the data volume *attachment*, but not the *volume* itself or the *Elastic IP*, more on that later. So move the resource definitions for `aws_instance.minecraft`, `aws_security_group.minecraft` and `aws_volume_attachment.minecraft` to `terraform/modules/ec2/main.tf`. 

Now we can reference the new module in our resource config like this:

{% highlight shell %}
module "ec2" {
  source = "modules/ec2"
}
{% endhighlight %}

> Whenever you reference a new module in your Terraform config you need to run `terraform init` before you will be able to apply changes.

If we try to use this module now Terraform will quickly inform us that we have some undefined references. We need to figure out what parameters to pass to our module before it will be truly useful. Our AMI data source and VPC definitions are outside of the EC2 module so any of those values will need to be passed in. It would also be nice to make the EC2 instance type variable, as well as the whitelisted IP addresses for SSH access. Let's create `variables.tf` to hold all of the inputs for our module.

{% highlight shell %}
variable "ami_id" {
  description = "AMI ID for instance"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type for instance"
  type        = string
}

variable "subnet_id" {
  description = "Subnet ID for instance"
  type        = string
}

variable "availability_zone" {
  description = "Subnet AZ for instance and SG"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID for security group"
  type        = string
}

variable "ssh_cidr_blocks" {
  description = "List of allowed CIDR blocks for SSH connections"
  type        = list(string)
}

variable "data_volume_id" {
  description = "ID of Minecraft world data volume to attach to instance"
  type        = string
}
{% endhighlight %}

Then we need to reference each of these variables in the `ec2` module.

{% highlight shell %}
resource "aws_instance" "minecraft" {
  ami           = var.ami_id
  instance_type = var.instance_type
  key_name      = "minecraft"

  vpc_security_group_ids = [aws_security_group.minecraft.id]

  availability_zone = var.availability_zone
  subnet_id         = var.subnet_id

  ...
}

resource "aws_security_group" "minecraft" {
  name        = "minecraft"
  description = "minecraft server"
  vpc_id      = var.vpc_id

  ingress {
    description = "ssh"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.ssh_cidr_blocks
  }

  ...
}

resource "aws_volume_attachment" "minecraft" {
  device_name = "/dev/sdf"
  volume_id   = var.data_volume_id
  instance_id = aws_instance.minecraft.id
}
{% endhighlight %}

And finally pass those parameters to the module:

{% highlight shell %}
module "ec2" {
  source = "modules/ec2"

  ami_id            = data.aws_ami.minecraft.id
  instance_type     = "m5a.large"
  availability_zone = "us-west-2a"
  subnet_id         = module.vpc.public_subnets[0]
  vpc_id            = module.vpc.vpc_id
  ssh_cidr_blocks   = ["77.191.142.21/32"]
  data_volume_id    = aws_ebs_volume.minecraft.id
}
{% endhighlight %}

Now we're ready to apply our changes, but nothing has actually changed about our infrastructure, we just refactored our configuration. However if we run `terraform init` then `terraform plan`, Terraform tells us that it plans to destroy 3 resources and create 3 new identical resources under a different namespace.

{% highlight shell %}
Plan: 3 to add, 1 to change, 3 to destroy.
{% endhighlight %}

This is not what we want. Instead we need to tell Terraform to move the existing resources to the new module namespace. We can do this with the `terraform state` command.

{% highlight shell %}
terraform state mv aws_instance.minecraft          module.ec2.aws_instance.minecraft
terraform state mv aws_security_group.minecraft    module.ec2.aws_security_group.minecraft
terraform state mv aws_volume_attachment.minecraft module.ec2.aws_volume_attachment.minecraft
{% endhighlight %}

If we run `terraform plan` again we see that indeed nothing has changed with our infrastructure and we've successfully refactored our EC2 resources into their own module.

{% highlight shell %}
No changes. Infrastructure is up-to-date.
{% endhighlight %}

### Using Published Modules

As easy as it is to create our own Terraform modules, much of the time we don't need to. We can get existing modules from a [variety of sources](https://www.terraform.io/docs/modules/sources.html) including the official [Terraform Registry](https://registry.terraform.io). Our VPC configuration is pretty basic so let's make use of Terraform's AWS VPC module instead. In `terraform/vpc.tf` get rid of all of the current resource definitions and replace them with the VPC module. We just need to configure it with the right AZ and public subnet, and tell it to create an Internet Gateway.

{% highlight shell %}
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "minecraft"
  cidr = "172.16.0.0/24"

  azs            = ["us-west-2a"]
  public_subnets = ["172.16.0.0/26"]

  create_igw = true
}
{% endhighlight %}

Run `terraform init` and `terraform plan`. Once again, Terraform wants to destroy and recreate the VPC. We need to use the `terraform state mv` command again to rename those resources.

{% highlight shell %}
terraform state mv aws_internet_gateway.main                module.vpc.aws_internet_gateway.this[0]
terraform state mv aws_route_table.public                   module.vpc.aws_route_table.public[0]
terraform state mv aws_route_table_association.usw2a_public module.vpc.aws_route_table_association.public[0]
terraform state mv aws_subnet.usw2a_public                  module.vpc.aws_subnet.public[0]
terraform state mv aws_vpc.main                             module.vpc.aws_vpc.this[0]
{% endhighlight %}

You may notice something else about the plan. `Plan: 8 to add, 1 to change, 7 to destroy.` Terraform wants to create another resource that doesn't exist, or at least Terraform doesn't know about it yet. Recall back in [Launching a Minimal Minecraft Server]({% post_url 2020-09-22-terraforming-minecraft-02 %}) when we created the Internet Gateway, AWS implicitly created a route for it. This route is explicitly defined in the Terraform VPC module so we need to import the existing resource into our Terraform state with `terraform import`. 

The `terraform import` command requires a resource name and identifier. The identifier can be the resource ID, ARN, or some combination of values depending on the resource. [The docs for the aws_route resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route) tell us that we need a combination of the route table ID and destination CIDR, so the import command looks like this:

{% highlight shell %}
terraform import module.vpc.aws_route.public_internet_gateway[0] rtb-0999a096f44bc9737_0.0.0.0/0

# module.vpc.aws_route.public_internet_gateway[0]: Importing from ID "rtb-0999a096f44bc9737_0.0.0.0/0"...
# module.vpc.aws_route.public_internet_gateway[0]: Import prepared!
#   Prepared aws_route for import
# module.vpc.aws_route.public_internet_gateway[0]: Refreshing state... [id=r-rtb-0999a096f44bc97371080289494]
{% endhighlight %}

Now if we run `terraform plan` again we see that, other than adding some default tags to the VPC resources, our infrastructure is unchanged.

> What else can you do with `terraform import`? Terraform plans to support generating configuration from existing resources in a future version, but for now, you can use a combination of `terraform import` and `terraform plan` commands to reverse-engineer your existing infrastructure. Say you have another EC2 instance you want to manage with Terraform. Start by creating a skeleton `aws_instance` resource definition with only the required parameters and import it. Then run `terraform plan` and Terraform will examine the current resource and compare it with the configuration for any changes. Rather than letting Terraform make the changes, update the resource definition using the information in the plan. Terraform has already done the discovery for you! When `terraform plan` no longer detects any changes you're done!

### Making a Testing Environment

With most of our resources in reusable modules it should be easy to spin up multiple copies of our infrastructure in other environments. Let's make a testing environment for our Minecraft project so we can play around with our configuration without breaking anything in production. I'm calling it *sandbox*.

{% highlight text %}
terraform/
|-- modules/
|-- sandbox/
|   `-- main.tf
`-- main.tf
{% endhighlight %}

Copy everything from `terraform/*.tf` to `terraform/sandbox/main.tf` *except* the resources `aws_dynamodb_table.pdizz_tflocks` and `aws_s3_bucket.pdizz_tfstate`. These are the shared [Terraform backend]({% post_url 2020-09-10-terraforming-minecraft-01 %}) for our whole project and we don't want conflicts from managing these resources in multiple places. We do need the AWS provider and S3 backend config. In the S3 backend config we defined the *key* `minecraft/terraform.tfstate` which is the location of our state files. We want our new environment to have it's own state so change the key to `minecraft/sandbox/terraform.tfstate` to match the directory structure of our project.

{% highlight shell %}
provider "aws" {
  region = "us-west-2"
}

terraform {
  backend "s3" {
    bucket         = "pdizz-tfstate"
    key            = "minecraft/sandbox/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "pdizz-tflocks"
    encrypt        = true
  }
}
{% endhighlight %}

In the `vpc` module we'll need to change the IP range for the VPC and subnet so they dont conflict with the current VPC. Also we can probably get away with a smaller `t3a.medium` EC2 instance since this is just a test environment.

{% highlight shell %}
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "minecraft-sb"
  cidr = "172.16.10.0/24"

  azs            = ["us-west-2a"]
  public_subnets = ["172.16.10.0/26"]

  create_igw = true
}

data "aws_ami" "minecraft" {
  owners           = ["self"]
  most_recent      = true

  filter {
    name   = "name"
    values = ["minecraft-*"]
  }
}

module "ec2" {
  source = "../modules/ec2"

  ami_id            = data.aws_ami.minecraft.id
  instance_type     = "t3a.medium"
  availability_zone = "us-west-2a"
  subnet_id         = module.vpc.public_subnets[0]
  vpc_id            = module.vpc.vpc_id
  ssh_cidr_blocks   = ["77.191.142.21/32"]
  data_volume_id    = aws_ebs_volume.minecraft.id
}
{% endhighlight %}

Since this is just a test environment, and we'll likely be destroying and recreating it often, it doesnt really need an Elastic IP. An EIP isn't very useful if it changes every time we create a new environment. We can just use the public IP of the EC2 instance instead, but there is a problem. Since we moved that resource to the `ec2` module we can no longer access that value directly. We need to create an output for the `ec2` module. In the `ec2` folder create the file `outputs.tf` and define an output for the instance.

{% highlight shell %}
output "instance" {
  value       = aws_instance.minecraft
  description = "Minecraft instance info"
}
{% endhighlight %}

Then in the `aws_route53_record` resource in `sandbox/main.tf` we can reference this output to get the public IP for the instance, and give our sandbox instance a unique hostname.

{% highlight shell %}
resource "aws_route53_record" "minecraft" {
    zone_id = "ZJO2WR3LOVAR9D"
    name    = "minecraft-sb.pdizz.com"
    type    = "A"
    ttl     = "300"
    records = [module.ec2.instance.public_ip]
}
{% endhighlight %}

> Don't forget to add the new host to `ansible/inventory.yaml` so we can configure the environment with our Ansible playbook. 

But what about the data volume? We could create a new one, but then each time we spin up this environment we would have to initialize the server, configure users, etc. Remember in [Preserving and Migrating Data with Volumes]({% post_url 2020-11-23-terraforming-minecraft-05 %}) I mentioned we could create a new volume from a snapshot of our production volume? First we need to define an output in our production config so we can get the volume ID in our sandbox environment.

{% highlight shell %}
output "volume" {
  value       = aws_ebs_volume.minecraft
  description = "data volume info"
}
{% endhighlight %}

Run `terraform apply` to generate the output in the state file. To access outputs from another Terraform state we can use the `terraform_remote_state` data source with the S3 backend. Then we can define a `aws_ebs_snapshot` resource based on that volume, and use it to create the new data volume for the sandbox environment.

{% highlight shell %}
# snapshot prod volume to build sandbox
data "terraform_remote_state" "prod" {
  backend = "s3"
  config = {
    bucket = "pdizz-tfstate"
    key    = "minecraft/terraform.tfstate"
    region = "us-west-2"
  }
}

resource "aws_ebs_snapshot" "minecraft_prod" {
  volume_id   = data.terraform_remote_state.prod.outputs.volume.id
  description = "prod snapshot for sandbox env data"

  tags = {
    Name = "minecraft-sb"
  }
}

resource "aws_ebs_volume" "minecraft" {
  availability_zone = "us-west-2a"
  size              = 20

  snapshot_id = aws_ebs_snapshot.minecraft_prod.id

  tags = {
    Name = "minecraft-sb"
  }
}
{% endhighlight %}

Now any time we want we can just run `terraform apply` and `ansible-playbook` to create an exact copy of our production environment in just a couple minutes! When we're finished, or if something breaks we can run `terraform destroy` to wipe the slate clean and start over.

> What to do about our production environment that is still sitting in the `terraform` folder? We could move our resources (except `aws_dynamodb_table` and `aws_s3_bucket` of course) to a new folder called `prod` to better organize things and separate our environment resources from shared backend resources. We would need to change the S3 backend key to `minecraft/prod/terraform.tfstate` and migrate our resources to this new path. To do that we would need to use the `terraform state mv` command again, but this time with the `-state=PATH` and `-state-out=PATH` options to tell Terraform to move these resources from the old state to the new state.

### Decoupling Modules and State

Since we can use variables and outputs to let our Terraform modules talk to each other we can effectively decouple our configuration into distinct components. For example, we could move the data volume and Elastic IP to a separate state which would allow us to tear down our other infrastructure without losing our data or giving up our IP address. 

The biggest advantage to decoupling our resources is limiting the *blast-radius* of potential bugs and breaking changes to our infrastructure. We can test changes in an entirely separate environment before ever touching production infrastructure, or change an application instance without affecting other resources or a shared VPC.

Unfortunately the downside is the overhead of managing resources in multiple locations. You have to switch to each root directory to run commands meaning more complexity in setting up new environments and automating deployments. Where and when to decouple environments and components of your infrastructure depends on the needs of your project and your team.