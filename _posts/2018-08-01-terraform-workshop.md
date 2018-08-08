---
layout: post
title: Terraform workshop
tags:
- terraform
---

I conducted a Terraform workshop internally for my org. I realized that most of
the Terraform tutorials try to emphasize more on the different types of resources
i.e. EC2, ELB, etc. while forgetting that the audience is one who has already
created such resources from the console and is looking for a guided best practice
approach for doing the same using Terraform.

Hence, instead of teaching users how to create X in Terraform, I flipped the 
approach to simulate an simple VPC setup through S3 i.e. Create S3 buckets 
which specify:

1. 1 VPC
2. 1 ELB with public subnet + 1 public security group
3. 2 instances behind ELB in private subnet + 1 private security group

This is faster to run, keeps focus on how to use Terraform rather than specifics
of different types of resources and also allows us to show resource dependency i.e. 
an ELB bucket is tagged with the VPC name - hence forcing Terraform to create VPC before ELB.

The approach seemed successfull and we could progress from a single-file, unmaintainable
Terraform setup in [assignment-1](https://github.com/saurabh-hirani/terraform-workshop)
to a clean, modular setup in [assignment-9](https://github.com/saurabh-hirani/terraform-workshop/tree/master/assignment-9).

Here is the material for the same:

1. Slides accompanying the workshop - [here](https://github.com/saurabh-hirani/talks/tree/master/terraform-workshop)
2. Workshop assignments - [here](https://github.com/saurabh-hirani/terraform-workshop)
3. Sample Terraform module created for the workshop - [here](https://github.com/saurabh-hirani/terraform-workshop-module)
