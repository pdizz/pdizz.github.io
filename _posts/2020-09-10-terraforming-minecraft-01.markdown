---
layout: post
title:  "Terraforming Minecraft - Intro and Project Setup"
date:   2020-09-10 0:00:00 -0700
categories: devops terraform aws minecraft
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

### Creating a Remote Backend for Terraform State

Rather than jumping in and launching servers we should take a moment to create a couple resources to support our Terraform project. Terraform keeps track of the resources it manages using state files. By default these state files are stored locally in your terraform project, but I would highly recommend configuring a remote backend for Terraform state, especially if you are working with a team. Terraform remote backends allow you to reliably share Terraform state with others and lock that state to prevent conflicts from multiple developers changing state concurrently.

> Check out Yevgeniy Brikman' excellent [A Comprehensive Guide to Terraform](https://blog.gruntwork.io/a-comprehensive-guide-to-terraform-b3d32832baca), particularly "How to manage Terraform state" and "How to use Terraform as a team" for a deeper look at these concepts.

Luckily, creating a remote backend in AWS is pretty simple. It requires only two resources: an Amazon S3 bucket to hold your state files and a DynamoDB table for the locks, and we can use Terraform to create these resources as well.

In your `main.tf` define an S3 bucket with versioning and encryption enabled and a DynamoDB table with a primary key of `LockID`

{% highlight text %}
    resource "aws_s3_bucket" "pdizz_tfstate" {
    bucket = "pdizz-tfstate"
    # Enable versioning so we can see the full revision history of our
    # state files
    versioning {
        enabled = true
    }
    # Enable server-side encryption by default
        server_side_encryption_configuration {
            rule {
                apply_server_side_encryption_by_default {
                    sse_algorithm = "AES256"
                }
            }
        }
    }
    
    resource "aws_dynamodb_table" "pdizz_tflocks" {
    name         = "pdizz-tflocks"
    billing_mode = "PAY_PER_REQUEST"
    hash_key     = "LockID"
        attribute {
            name = "LockID"
            type = "S"
        }
    }
{% endhighlight %}

Run `terraform apply` to create the resources. We haven't yet told Terraform to use the remote backend so it will create state files locally in the `.terraform` directory.

{% highlight text %}
    ...

    Plan: 2 to add, 0 to change, 0 to destroy.

    Do you want to perform these actions?
    Terraform will perform the actions described above.
    Only 'yes' will be accepted to approve.

    Enter a value: yes

    aws_dynamodb_table.pdizz_tflocks: Creating...
    aws_s3_bucket.pdizz_tfstate: Creating...
    aws_s3_bucket.pdizz_tfstate: Creation complete after 6s [id=pdizz-tfstate]
    aws_dynamodb_table.pdizz_tflocks: Creation complete after 10s [id=pdizz-tflocks]

    Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
{% endhighlight %}

Now in `main.tf` you can define the Terraform backend with your S3 bucket and DynamoDB table. The `key` parameter is used for the path within your S3 bucket where the state will be stored. You can use this to organize and store state for multiple environments or projects in the same bucket.

{% highlight text %}
    terraform {
        backend "s3" {
            bucket         = "pdizz-tfstate"
            key            = "minecraft/terraform.tfstate"
            region         = "us-west-2"
            dynamodb_table = "pdizz-tflocks"
            encrypt        = true
        }
    }
{% endhighlight %}

In your project root run `terraform init`. Terraform will ask if you want to migrate your current state to the new remote backend. Type "yes" and your Terraform state is now managed in AWS.
