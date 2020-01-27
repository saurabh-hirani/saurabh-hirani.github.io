---
layout: post
title: Terraform AWS state management module
tags:
- AWS, terraform
---

I wrote a Terraform module to manage Terraform remote state infra uniformly
[here](https://github.com/saurabh-hirani/terraform-aws-state-mgmt).

This module can be used as a first step to create AWS S3 state buckets with various
features like versioning, logging, DynamoDB table support, etc. in a standard way as
opposed to creating them manually.
