---
layout: post
title:  "Provisioning a Minecraft Server with Terraform - AWS Setup"
date:   2020-09-10 0:00:00 -0700
categories: devops terraform aws
series: minecraft-terraform
order: 1
---

In this series I want to explore techniques for managing software infrastructure in AWS by building a mincraft server. I'm hoping it will be a fun exercize, but the challenges we run into by building, maintaining and upgrading a minecraft server in the cloud mirror many of the same challenges found in other software projects. Crafting reliable ifrastructure-as-code, upgrading software and infrastructure, deploying changes, maintaining availability are challenges common to many other software projects and the techniques we learn here can be applied to projects in many other domains.

### Creating an IAM User for Terraform

While you can use your main AWS account credentials to manage all of your infrastructure it is **highly** recommended to create a non-administrative user with only the permissions necessary to do the job. It is a good idea to do this early since it will also come in to play when we need to separate our infrastructure into different environments. Not only can we create a logical separation between environments but we can also enforce that separation using IAM's own ACL.

ACL configuration in AWS is very, very... *very* granular, but rather than go too deep into specifics we'll start by creating a system user with full access to just the services required to build our infrastructure. In the IAM > Users view click "Add user" and give the user a descriptive name like "tf_user" (naming things is hard!). Choose "Programatic access" only (this user will not be able to log in to AWS, but only access services using a generated key) and click next.

For permissions either create a group or attach directly the following pre-defined policies:
- AmazonEC2FullAccess
- AmazonS3FullAccess
- AmazonDynamoDBFullAccess
- AmazonRoute53FullAccess

Finish creating the user and copy or download the user's access key, then create a file in your home directory `~/.aws/credentials` with the contents:

{% highlight text %}
    [default]
    aws_access_key_id = AWSACCESSKEYID
    aws_secret_access_key = thesecretkeygoeshere
{% endhighlight %}

### Initializing a Terraform Project

Create a file `main.tf` at the root of your project directory and define the AWS provider with your chosen region.

{% highlight text %}
    provider "aws" {
        region = "us-west-2"
    }
{% endhighlight %}

In your project directory run `terraform init` to install the AWS provider plugins and initialize your project.

{% highlight text %}
    Initializing provider plugins...
    - Finding latest version of hashicorp/aws...
    - Installing hashicorp/aws v3.7.0...
    - Installed hashicorp/aws v3.7.0 (signed by HashiCorp)
{% endhighlight %}

Now we're ready to start building AWS resources with Terraform.
