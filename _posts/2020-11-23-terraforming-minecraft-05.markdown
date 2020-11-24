---
layout: post
title:  "Terraforming Minecraft - Preserving and Migrating Data with Volumes"
date:   2020-11-23 0:00:00 -0700
categories: devops packer terraform aws minecraft
series: minecraft-terraform
order: 5
---

The inevitable happened and a new Minecraft Server version was released. Fortunately the upgrade process for Minecraft is pretty simple. All we need to do is install the latest server.jar and Minecraft will handle the rest the first time it starts. We could just install it on the instance or in the Ansible playbook, but refering back to [Building Custom AMIs with Packer]({% post_url 2020-10-01-terraforming-minecraft-03 %}), we want to avoid doing installs in configuration management. 

We need to go back to the start of our pipeline and build a new AMI with the latest server.jar, which means creating a new EC2 instance for the server. By now you may have become rather attached to your Minecraft World like I have and would hate to see it destroyed. So what do we do with our World data? AWS provides a solution for this with EBS volumes.

### Storing Minecraft Data on a Data Volume

An EBS volume is a disk that can be attached to an EC2 instance. We can store our Minecraft World data (and anything else we want) on the volume and easily migrate it from one instance to another. Lets start by defining a Terraform resource in `ec2.tf` for our data volume and an attachment to the current instance.

{% highlight shell %}
resource "aws_ebs_volume" "minecraft" {
  availability_zone = aws_subnet.usw2a_public.availability_zone
  size              = 20

  tags = {
    Name = "minecraft"
  }

  # DONT DELETE THE DATA VOLUME!
  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_volume_attachment" "minecraft" {
  device_name = "/dev/sdf"
  volume_id   = aws_ebs_volume.minecraft.id
  instance_id = aws_instance.minecraft.id
}
{% endhighlight %}

Terraform will create a new 20GB volume and attach it to our instance on the device `/dev/sdf` which may translate to a different device like `/dev/nvme1n1` on the instance. [Before we can use a new volume](https://docs.amazonaws.cn/en_us/AWSEC2/latest/UserGuide/ebs-using-volumes.html) it needs a filesystem and it needs to be mounted to a directory in our instance. Lets use Ansible to configure this. Create a new role `ansible/roles/volume/tasks/main.yaml`

{% highlight yaml %}
- name: filesystem exists on volume device
  filesystem:
    dev: /dev/nvme1n1
    fstype: ext4

- name: minecraft data directory exists
  file:
    path: /mnt/minecraft
    state: directory
    owner: minecraft
    group: minecraft
    mode: 0755

- name: minecraft data directory mounted
  mount:
    path: /mnt/minecraft
    src: /dev/nvme1n1
    fstype: ext4
    opts: noatime,relatime,nodiratime
    state: mounted
{% endhighlight %}

Don't forget to include the new role in `ansible/playbook.yaml`

{% highlight yaml %}
---
- hosts: minecraft
  become: yes
  become_user: root

  roles:
    - volume
    - minecraft
{% endhighlight %}

Run the playbook and Ansible will ensure that the volume is mounted and ready for our migration, but first we have to decide what to migrate.

### Configuration vs. Data

Some applications separate config files and data into distinct directories, making it easy to decouple them. Unfortunately, Minecraft just kinda puts everything together in one place. Ideally we want to avoid any conflict between files managed by Ansible and user data managed by the Minecraft Server. We could let Ansible configure `server.properties` to make it easier to deploy pre-configured servers, but then Ansible would overwrite any changes made by users any time we run the playbook. We also dont want any binaries like `server.jar` that would replace the installed version on the AMI when we move the volume to a new instance.

So, what files to store on the data volume? I went with the `world/` directory of course, along with anything else that is created or updated by the server, like `logs/`, and files that are updated by user commands like `server.properties` and `ops.json`. This is what I chose to migrate:

- banned-ips.json
- banned-players.json
- logs/
- ops.json
- server.properties
- usercache.json
- whitelist.json
- world/

With that settled we're ready to do our one-time migration to the data volume. Be sure to stop your Minecraft Server `systemctl stop minecraft` then move each directory and file to `/mnt/minecraft/`. Finally we need symlinks in `/opt/minecraft` to point to their new locations so the Minecraft Server can find them. In the Ansible role `ansible/roles/volume/tasks/main.yaml` add another task to define the symlinks.

{% highlight yaml %}
- name: server state files linked to volume
  file:
    src: /mnt/minecraft/{{ item }}
    dest: /opt/minecraft/{{ item }}
    state: link
    owner: minecraft
    group: minecraft
  with_items:
    - banned-ips.json
    - banned-players.json
    - logs
    - ops.json
    - server.properties
    - usercache.json
    - whitelist.json
    - world
{% endhighlight %}

Run the playbook again and the migration is complete. Now we can easily move our Minecraft World data to a new instance.

> Now would be a good time to stop your server process again and take a snapshot of your data volume. This snapshot can be used to restore the original version if something goes wrong.

### Promoting a New Instance

With that done we can fire up our pipeline and deploy a new instance with the latest Minecraft version. First we have to build a new AMI. Update the `packer/scripts/minecraft-install.sh` script with the URL for the latest Minecraft version.

{% highlight shell %}
sudo useradd --shell /bin/false --home-dir /opt/minecraft minecraft

sudo yum -y install wget java screen
sudo wget -qO /opt/minecraft/server.jar https://launcher.mojang.com/v1/objects/35139deedbd5182953cf1caa23835da59ca3d7cd/server.jar # 1.16.4

sudo chown -R minecraft.minecraft /opt/minecraft
{% endhighlight %}

Then run the packer command and increment the `build_number` to build a new AMI. Once the image is complete, switch to the `terraform` directory and run `terraform apply`. The `aws_ami` data source will pull the id of the latest AMI and Terraform lets us know that, because the AMI chagned, the current instance will need to be replaced.

{% highlight shell %}
Terraform will perform the following actions:

  # aws_instance.minecraft must be replaced
-/+ resource "aws_instance" "minecraft" {
      ~ ami = "ami-0b560b8f3b4b4cb9b" -> "ami-0c2beb146961fcbf5" # forces replacement
      ...
    }

  # aws_volume_attachment.minecraft must be replaced
-/+ resource "aws_volume_attachment" "minecraft" {
        device_name = "/dev/sdf"
      ~ id          = "vai-2145474463" -> (known after apply)
      ~ instance_id = "i-0a97ad212addfc233" -> (known after apply) # forces replacement
        volume_id   = "vol-0e6f1ebdf7b8fce1f"
    }
{% endhighlight %}

Notice that `aws_instance.minecraft` will be replaced, along with the attachment resource `aws_volume_attachment.minecraft`, but most importantly **the volume_id does not change**. Our World data volume will be attached to the new instance. Type "yes" and Terraform will go to work creating the new instance and destroying the old one.

Finally we need to run the Ansible playbook to configure the new instance. Ansible will ensure that the symlinks exist for the files in the data volume, and that the server daemon is configured and started. When the Minecraft Server starts it will make the necessary changes to the World files and the upgrade is done!

### What Else Can We Do with Volumes?

> We've seen that we can easily migrate our data between EC2 instances, and we can create point-in-time snapshots of our data volume. From a snapshot we can create any number of new volumes to restore data, or deploy clones of our application on new instances. We could even attach the same volume to multiple instances if our application supports it.

So, worst case scenario, the upgrade failed and data is corrupted! We need to restart our pipeline with the original AMI and roll back to a working version immediately. We could create a new volume from the snapshot and import it into Terraform state (more on import and Terraform state later). Delete or rename the new (failed) AMI so Terraform will use the original version. Run `terraform apply` and finally the Ansible playbook and we've quickly reverted to the last working state of our server.

Phew! We should probably be more proactive and try to avoid that whole situation by testing the upgrade in another environment. Before running the upgrade in production we could create a volume from our snapshot along with a new EC2 instance using the new AMI.

{% highlight shell %}
resource "aws_instance" "minecraft_test" {
  ami           = data.aws_ami.minecraft.id
  instance_type = "t3a.medium"
  key_name      = "minecraft"
  ...
}

resource "aws_ebs_snapshot" "minecraft_test" {
  volume_id   = aws_ebs_volume.minecraft.id
  description = "pre-upgrade snapshot 2020-11-23"

  tags = {
    Name = "minecraft"
  }
}

resource "aws_ebs_volume" "minecraft_test" {
  availability_zone = aws_subnet.usw2a_public.availability_zone
  size              = 20
}

resource "aws_volume_attachment" "minecraft_test" {
  device_name = "/dev/sdf"
  volume_id   = aws_ebs_volume.minecraft_test.id
  instance_id = aws_instance.minecraft_test.id
}
{% endhighlight %}

If that works we can be confident going forward with our upgrade in production.
