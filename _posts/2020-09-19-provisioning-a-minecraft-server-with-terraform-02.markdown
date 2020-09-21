---
layout: post
title:  "Provisioning a Minecraft Server with Terraform - Resources for Terraform State"
date:   2020-09-19 0:00:00 -0700
categories: devops terraform aws
series: minecraft-terraform
order: 2
---

Rather than jumping in and launching servers we should take a moment to create a couple resources to support our Terraform project. Terraform keeps track of the resources it manages using state files. By default these state files are stored locally in your terraform project, but I would highly recommend configuring a remote backend for Terraform state, especially if you are working with a team. Terraform remote backends allow you to reliably share Terraform state with others and lock that state to prevent conflicts from multiple developers changing state concurrently.

> Check out Yevgeniy Brikman' excellent [A Comprehensive Guide to Terraform](https://blog.gruntwork.io/a-comprehensive-guide-to-terraform-b3d32832baca), particularly "How to manage Terraform state" and "How to use Terraform as a team" for a deeper look at these concepts.

### Creating a Remote Backend for Terraform State

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
