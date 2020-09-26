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

In addition we also need to define VPC Subnets which are IP blocks for your network that provide public or private IP addresses for the specific Availability Zones in your region. Reference your VPC id `aws_vpc.main.id` in your subnets. We can start with just one public subnet in us-west-2a for now.

{% highlight text %}
    resource "aws_subnet" "usw2a_public" {
        vpc_id     = aws_vpc.main.id
        cidr_block = "172.33.0.0/20"
        map_public_ip_on_launch = true
        
        tags = {
            Name = "usw2a-public"
        }
}   
{% endhighlight %}

Run `terraform apply` and review and confirm the plan and Terraform will create the VPC and Subnet in your region.

{% highlight text %}
    Terraform will perform the following actions:

    # aws_subnet.usw2a_public will be created
    + resource "aws_subnet" "usw2a_public" {
        + arn                             = (known after apply)
        + assign_ipv6_address_on_creation = false
        + availability_zone               = (known after apply)
        + availability_zone_id            = (known after apply)
        + cidr_block                      = "172.33.0.0/20"
        + id                              = (known after apply)
        + ipv6_cidr_block_association_id  = (known after apply)
        + map_public_ip_on_launch         = true
        + owner_id                        = (known after apply)
        + tags                            = {
            + "Name" = "usw2a-public"
            }
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

That is technically all you need to start launching EC2 instances, but before you can actually connect to anything from the internet you need to set up a route to your VPC from the internet. The most basic configuration to accomplish this is with an internet gateway and a single public subnet. 

First define an internet gateway resource attached to your VPC, then a route table with a route for your internet gateway with a CIDR block of `0.0.0.0/0`. Finally create an association resource to attach the route table to your public subnet, which will automatically create the local route to your subnet.

{% highlight text %}
    resource "aws_internet_gateway" "main" {
        vpc_id = aws_vpc.main.id

        tags = {
            Name = "main"
        }
    }

    resource "aws_route_table" "public" {
        vpc_id = aws_vpc.main.id

        route {
            cidr_block = "0.0.0.0/0"
            gateway_id = aws_internet_gateway.main.id
        }

        tags = {
            Name = "public"
        }
    }

    resource "aws_route_table_association" "usw2a_public" {
        subnet_id      = aws_subnet.usw2a_public.id
        route_table_id = aws_route_table.public.id
    }
{% endhighlight %}

Run `terraform apply` again to finish setting up your VPC.

{% highlight text %}
    Terraform will perform the following actions:

    # aws_internet_gateway.main will be created
    + resource "aws_internet_gateway" "main" {
        + arn      = (known after apply)
        + id       = (known after apply)
        + owner_id = (known after apply)
        + tags     = {
            + "Name" = "main"
            }
        + vpc_id   = "vpc-05644ed8f077713b8"
        }

    # aws_route_table.public will be created
    + resource "aws_route_table" "public" {
        + id               = (known after apply)
        + owner_id         = (known after apply)
        + propagating_vgws = (known after apply)
        + route            = [
            + {
                + cidr_block                = "0.0.0.0/0"
                + egress_only_gateway_id    = ""
                + gateway_id                = (known after apply)
                + instance_id               = ""
                + ipv6_cidr_block           = ""
                + local_gateway_id          = ""
                + nat_gateway_id            = ""
                + network_interface_id      = ""
                + transit_gateway_id        = ""
                + vpc_peering_connection_id = ""
                },
            ]
        + tags             = {
            + "Name" = "public"
            }
        + vpc_id           = "vpc-05644ed8f077713b8"
        }

    # aws_route_table_association.usw2a_public will be created
    + resource "aws_route_table_association" "usw2a_public" {
        + id             = (known after apply)
        + route_table_id = (known after apply)
        + subnet_id      = "subnet-027e907b5f2bfe0aa"
        }

    Plan: 3 to add, 0 to change, 0 to destroy.
{% endhighlight %}

Now you can *finally* launch an EC2 instance that is accessible from the internet.
