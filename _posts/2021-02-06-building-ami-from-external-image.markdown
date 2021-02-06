---
layout: post
title:  "Building a Custom AMI from an Existing Image"
date:   2021-02-06 0:00:00 -0700
categories: devops packer terraform aws
---

In [Terraforming Minecraft - Building Custom AMIs with Packer]({% post_url 2020-10-01-terraforming-minecraft-03 %}) we used Packer's `amazon-ebs` builder to create custom AMIs from a source AMI (CentOS minimal) for launching EC2 instances. This is really convenient *if you're keeping everything in AWS*, but what if you want to build images for other providers, or you already have an existing image like an OVA? To build an AMI from a local image we can use the `amazon-import` post-processor.

### Importing an Existing Image as an AWS AMI

Note that `amazon-import` is a *post-processor* not a builder. To use it we need to configure a builder that will export the image as an OVA, then `amazon-import` will import the resulting artifact into AWS. The process works like this:

- An OVA is exported after builders and provisioners have run
- Packer uploads the OVA temporarily to an S3 bucket where the AWS VM Import/Export service can access it
- The VM Import/Export service converts the OVA into an AMI in your account
- Packer deletes the uploaded OVA from the S3 bucket

In order for this process to work we need to provision a few resources in our AWS account.

### Configuring an AWS S3 Bucket for Image Uploads

First we'll need an S3 bucket for uploading our image. Using Terraform let's define an `aws_s3_bucket` resource. Since these uploads are temporary this bucket should cost almost nothing. However, we should add a basic `lifecycle_rule` to expire any object left in the bucket after 1 day, in case the process fails before Packer can clean up.

{% highlight shell %}
# bucket for uploading OVAs for AMI import
resource "aws_s3_bucket" "image_builder" {
  bucket = "pdizz-image-builder"

  versioning {
    enabled = true
  }

  # in case packer doesn't clean up uploaded OVAs if the build fails
  lifecycle_rule {
    id      = "cleanup"
    enabled = true

    expiration {
      days = 1
    }
  }

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}
{% endhighlight %}

Then we need an IAM policy to allow our Packer user to access the S3 bucket and EC2 resources.

{% highlight shell %}
resource "aws_iam_policy" "image_builder" {
  name        = "image_builder"
  description = "allow packer builder user to upload ova's to the images bucket, and import/export images in ec2"
  path        = "/"

  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListAllMyBuckets"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:CreateBucket",
        "s3:DeleteBucket",
        "s3:DeleteObject",
        "s3:GetBucketLocation",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:PutObject"
      ],
      "Resource": ["arn:aws:s3:::pdizz-image-builder/*"]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CancelConversionTask",
        "ec2:CancelExportTask",
        "ec2:CreateImage",
        "ec2:CreateInstanceExportTask",
        "ec2:CreateTags",
        "ec2:DeleteTags",
        "ec2:DescribeConversionTasks",
        "ec2:DescribeExportTasks",
        "ec2:DescribeInstanceAttribute",
        "ec2:DescribeInstanceStatus",
        "ec2:DescribeInstances",
        "ec2:DescribeTags",
        "ec2:ImportInstance",
        "ec2:ImportVolume",
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:TerminateInstances",
        "ec2:ImportImage",
        "ec2:ImportSnapshot",
        "ec2:DescribeImportImageTasks",
        "ec2:DescribeImportSnapshotTasks",
        "ec2:CancelImportTask",
        "ec2:DescribeImageAtrribute",
        "ec2:DescribeImages"
      ],
      "Resource": "*"
    }
  ]
}
EOF
}
{% endhighlight %}

This policy can then be attached to a group, role or IAM user that Packer will use.

> This is optional, of course. If you're just using your admin user credentials to do everything you will already be able to access everything in this policy, but it's a bad idea. Get used to creating limited roles and policies for your automated processes and follow the Principle of Least Privilege.

### Allowing AWS to Access your Bucket and EC2

Now that we can upload an image to S3, AWS needs permission to access this image and create an AMI for us. First we need a role called `vmimport` that the VM Import/Export service can assume to access our account.

{% highlight shell %}
resource "aws_iam_role" "vmimport" {
  name = "vmimport"
  description = "AWS VM Import/Export service will use this role to access your resources"

  assume_role_policy = <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "vmie.amazonaws.com"
            },
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringEquals": {
                    "sts:Externalid": "vmimport"
                }
            }
        }
    ]
}
EOF
}
{% endhighlight %}

> By default Packer tells AWS to look for a role named `vmimport`. If for some reason you need to use a different name for the role you need to specify that in the `amazon-import` post-processor with the `role_name` option.

Next we need to create a policy for this role that will allow the VM Import/Export service to access the images bucket and create AMIs in our account. Define an `aws_iam_policy` resource and an `aws_iam_role_policy_attachment` to link the policy to the `vmimport` role.

{% highlight shell %}
resource "aws_iam_policy" "vmimport" {
  name        = "vmimport"
  path        = "/"
  description = "allow AWS VM Import/Export service to access uploads in the images bucket and create images in ec2"

  policy = <<EOF
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Action":[
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource":[
        "arn:aws:s3:::pdizz-image-builder"
      ]
    },
    {
      "Effect":"Allow",
      "Action":[
        "s3:GetObject"
      ],
      "Resource":[
        "arn:aws:s3:::pdizz-image-builder/*"
      ]
    },
    {
      "Effect":"Allow",
      "Action":[
        "ec2:ModifySnapshotAttribute",
        "ec2:CopySnapshot",
        "ec2:RegisterImage",
        "ec2:Describe*"
      ],
      "Resource":"*"
    }
  ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "vmimport" {
  role       = aws_iam_role.vmimport.name
  policy_arn = aws_iam_policy.vmimport.arn
}
{% endhighlight %}

Finally we can run `terraform apply` to provision these resources and we're ready to start building!

### Building an AMI from an OVA

Now all we need to do is plug in the `amazon-import` post-processor after any builder that produces an OVA artifact like `virtualbox-iso`. Configure the post-processor with your region, S3 bucket, and any tags you want.

{% highlight json %}
{
  "variables": {
    "name": null,
    "build_number": null
  },
  "builders": [
    {
      "type": "virtualbox-iso",
      "format": "ova",
      ...
    }
  ],
  "provisioners": [
    ...
  ],
  "post-processors": [
    {
      "s3_key_name": "my-image.ova",
      "type": "amazon-import",
      "region": "us-west-2",
      "s3_bucket_name": "pdizz-image-builder",
      "tags": {
        "Name": "{%raw%}{{user `name`}}{%endraw%}",
        "build": "{%raw%}{{user `build_number`}}{%endraw%}"
      }
    }
  ]
}
{% endhighlight %}

If you already have an OVA and you just want to import it as-is (without any extra provisioning), you can use the `file` builder. Rather than booting up the image and exporting it as a new OVA it will just copy the existing file.

{% highlight json %}
{
  "variables": {
    "name": null,
    "build_number": null
  },
  "builders": [
    {
      "type": "file",
      "source": "my-image.ova",
      "target": "/tmp/my-image.ova"
    }
  ],
  "post-processors": [
    {
      "s3_key_name": "my-image.ova",
      "type": "amazon-import",
      "region": "us-west-2",
      "s3_bucket_name": "pdizz-image-builder",
      "tags": {
        "Name": "{%raw%}{{user `name`}}{%endraw%}",
        "build": "{%raw%}{{user `build_number`}}{%endraw%}"
      }
    }
  ]
}
{% endhighlight %}

And that's it! The VM Import/Export process can take a while, 15-20 minutes or more, so be patient. You can check the status of the import task with the command `aws ec2 describe-import-image-tasks --import-task-ids <import-task-id>` using the import task id found in the output from Packer.
