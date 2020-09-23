---
layout: post
title:  "Provisioning a Minecraft Server with Terraform - Launching a VPC"
date:   2020-09-22 0:00:00 -0700
categories: devops terraform aws
series: minecraft-terraform
order: 3
---

The first, most essential resource for any application in AWS is a VPC, or virtual network that will contain all your application resources within your chosen region.

### Creating a VPC and subnets

While you can define all your terraform resources in the same file, it is usually better to break them up into logical pieces for your own sanity. Create a file in the same directory as your `main.tf` called `vpc.tf` and define your first (default) VPC.

{% highlight text %}
    resource "aws_vpc" "main" {
        cidr_block       = "172.33.0.0/16"
        instance_tenancy = "default"
    }
{% endhighlight %}

In addition we also need to define VPC Subnets which are IP blocks for your network that provide public or private IP addresses for the specific Availability Zones in your region. Reference your VPC id `aws_vpc.main.id` in your subnets. We can start with just one subnet in us-west-2a for now.

{% highlight text %}
    resource "aws_subnet" "usw2a" {
        vpc_id                  = aws_vpc.main.id
        availability_zone       = "us-west-2a"
        cidr_block              = "172.33.0.0/20"
        map_public_ip_on_launch = true
    }
{% endhighlight %}

Run `terraform apply` and review and confirm the plan and Terraform will create the VPC and Subnet in your region.

{% highlight text %}
    # aws_subnet.usw2a will be created
    + resource "aws_subnet" "usw2a" {
        + arn                             = (known after apply)
        + assign_ipv6_address_on_creation = false
        + availability_zone               = "us-west-2a"
        + availability_zone_id            = (known after apply)
        + cidr_block                      = "172.31.0.0/20"
        + id                              = (known after apply)
        + ipv6_cidr_block_association_id  = (known after apply)
        + map_public_ip_on_launch         = true
        + owner_id                        = (known after apply)
        + vpc_id                          = (known after apply)
        }

    # aws_vpc.main will be created
    + resource "aws_vpc" "main" {
        + arn                              = (known after apply)
        + assign_generated_ipv6_cidr_block = false
        + cidr_block                       = "172.33.0.0/16"
        + default_network_acl_id           = (known after apply)
        + default_route_table_id           = (known after apply)
        + default_security_group_id        = (known after apply)
        + dhcp_options_id                  = (known after apply)
        + enable_classiclink               = (known after apply)
        + enable_classiclink_dns_support   = (known after apply)
        + enable_dns_hostnames             = (known after apply)
        + enable_dns_support               = true
        + id                               = (known after apply)
        + instance_tenancy                 = "default"
        + ipv6_association_id              = (known after apply)
        + ipv6_cidr_block                  = (known after apply)
        + main_route_table_id              = (known after apply)
        + owner_id                         = (known after apply)
        }

    Plan: 2 to add, 0 to change, 0 to destroy.
{% endhighlight %}
