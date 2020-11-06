---
layout: post
title:  "Terraforming Minecraft - Building Custom AMIs with Packer"
date:   2020-10-01 0:00:00 -0700
categories: devops packer terraform aws minecraft
series: minecraft-terraform
order: 3
---

At this point we have a working Minecraft server built with Terraform. More importantly, we already have all of the resources we need, defined in our Terraform project, to create any number of Minecraft servers. In fact we could launch fully on-demand Minecraft servers at any time with the push of a button, right? Well, there are a few glaring issues with oure current approach that need to be addressed.

> Software is constantly changing. New versions are released and old versions are deprecated. The softare you rely on today may not be available tomorrow. There is no guarantee that that your install script will work every time, and eventually it will stop working all together.

### Why Build Images?

It's simply not feasible to to install all of the software for our application every time we launch a new instance. Each new instance might have slightly different installed versions leading to insidious bugs that are very difficult to reproduce. 

Softare versions can be removed from repositories, or the repositories themselves might temporarily unavailable causing the new instance to fail completely. Imagine a code-red situation where you need to replace a dead instance, only to have it fail because a particular package version is no longer available.

The installation process can be very time consuming. While launching a new instance might take mere seconds, it could be many minutes until it is actually ready, causing delays in scaling or restoring service during an outage.

The solution is to capture our application environment in a known working state into an image we can use to easily replicate that environment at any time.

### Building AMIs with Packer

Packer is a tool for creating machine images. Packer works by using a pre-existing image to launch a virtual machine, then runs commands to make changes to that environment. Then it repackages that state into a new image. While all the steps to create an AMI (Amazon Machine Image) can be done manually through the console, Packer can automate the process and make it repeatable. Packer can also build identical images for other VM and cloud providers at the same time.

So to build a custom AMI for our Minecraft server we can simply start with the minimal CentOS AMI we're already using, run the installation script, and build a new AMI. Create a new directory in your project root called `images` or something for your image build scripts and create a packer file called `ami-build.json`. You'll need to tell Packer to use the `amazon-ebs` builder and provide the `source_ami` id and an ec2 instance type so it can launch a temporary EC2 instance on your behalf. Then you need to configure a `shell` provisioner to run your installation script once the machine is running.

{% raw %}
    {
        "builders": [
            {
                "type": "amazon-ebs",
                "region": "us-west-2",
                "source_ami": "ami-0bc06212a56393ee1",
                "instance_type": "t3a.micro",
                "ssh_username": "centos",
                "ssh_keypair_name": "minecraft_build",
                "ami_name": "my-minecraft-ami",
                "ena_support": "true",
                "ssh_private_key_file": "~/minecraft_build.pem",
                "tags": {
                    "Name": "minecraft",
                    "Description": "minecraft server image"
                }
            }
        ],
        "provisioners": [
            {
                "type": "shell",
                "execute_command": "echo 'centos' | {{.Vars}} sudo -S -E bash '{{.Path}}'",
                "scripts": [
                    "images/scripts/minecraft-install.sh"
                ]
            }
        ]
    }

{% endraw %}

And the installation script `build/scripts/minecraft-install.sh`...

{% highlight text %}
    # create minecraft user with no login
    sudo useradd --shell /bin/false --home-dir /opt/minecraft minecraft

    sudo yum -y install wget java
    sudo wget -qO /opt/minecraft/server.jar https://launcher.mojang.com/v1/objects/f02f4473dbf152c23d7d484952121db0b36698cb/server.jar # 1.16.3

    sudo echo "eula=true" > /opt/minecraft/eula.txt
{% endhighlight %}

From your project root run `packer build images/packer/ami-build.json` and packer will launch an EC2 instance from the CentOS AMI, run the install script, and create a new AMI.

{% highlight text %}
    ==> amazon-ebs: Waiting for instance (i-07bd7b60ef591d691) to become ready...
    ==> amazon-ebs: Waiting for SSH to become available...
    ==> amazon-ebs: Connected to SSH!
    ==> amazon-ebs: Provisioning with shell script: build/scripts/minecraft-install.sh
    ...
    ==> amazon-ebs: Stopping the source instance...
    ...
    Build 'amazon-ebs' finished.

    ==> Builds finished. The artifacts of successful builds are:
    --> amazon-ebs: AMIs were created:
    us-west-2: ami-04e29ccba116723ce
{% endhighlight %}

It would be nice to have some way to keep track of our AMIs with an id or build number, and to be able to choose different source AMIs so lets add a couple variables to our packer file and use those values for our `source_ami`, `ami_name` and `tags`.

{% raw %}
    {
        "variables": {
            "build_number": "",
            "source_ami": ""
        },
        "builders": [
            {
                "type": "amazon-ebs",
                "region": "us-west-2",
                "source_ami": "{{user `source_ami`}}",
                "instance_type": "t3a.micro",
                "ssh_username": "centos",
                "ssh_keypair_name": "minecraft_build",
                "ami_name": "minecraft-{{user `build_number`}}",
                "ena_support": "true",
                "ssh_private_key_file": "~/minecraft_build.pem",
                "tags": {
                    "Name": "minecraft",
                    "build": "{{user `build_number`}}",
                    "Description": "minecraft build {{user `build_number`}}"
                }
            }
        ],
        "provisioners": [
            {
                "type": "shell",
                "execute_command": "echo 'centos' | {{.Vars}} sudo -S -E bash '{{.Path}}'",
                "scripts": [
                    "build/scripts/minecraft-install.sh"
                ]
            }
        ]
    }
{% endraw %}

Now you can specify different source AMIs to build from and keep track of your custom AMIs with a build number. How you choose to do that is up to you.

### Strategies for Managing Images

There are two distinct strategies we can use to apply changes to our images. There are advantages and disadvantages to each method, and both methods can even be combined into a hybrid approach, but which method is the best for your project is up to you. 

#### The Incremental Method

The incremental method is to always base the new image on the previous version and only worry about one small change at a time. We've already seen that Packer can use any source AMI, including any of your custom AMIs, to build a new image. The advantage to this is that it can be very fast to develop, test, and deploy a single small change like this. The disadvantage to this method is that, because you are only concerned with the change from one version to the next, you lose the ability to completely recreate the image from scratch if necessary. 

Changes and mistakes can compound over time in what I call *configuration drift*. The longer you maintain a system the more chances the gremlins have to go to work on it, or maybe I'm just superstitious... While you can generally roll back a few versions to a previous image, it's never a guarantee. You may discover some update several versions ago caused a bug that has been hiding out ever since. By that time your application may have changed enough that it wont work on the last known good image.

#### The Pipeline Method

The method I prefer is to always build new images from a pristine source AMI, usually just a minimal OS installation, and install all your dependencies from scratch each time. Of course that means you have to maintain the code for the entire installation process. Although the situation is far less critical than in an outage, you can run into the same problem with dependencies being unavailable causing delay and extra work getting a new image built. The upside is that because you are "starting fresh" with every image there is no chance for drift. 

With this method you can build a base image with common software packages installed, and use that to build images for different application environments like pre-production and production that have may have some extra packages but still come from the same base image. You can even build images for multiple different cloud providers at once from a single source image with more certainty that they will be consistent in all providers

> Many projects will benefit from a combination of these two methods. For example a pipeline to build major versions of your image from a clean source, but with small incremental updates before the next major image is built.
