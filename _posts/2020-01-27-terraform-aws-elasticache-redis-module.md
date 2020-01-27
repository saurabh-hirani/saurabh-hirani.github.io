---
layout: post
title: Terraform AWS Elasticache Redis module for multiple setups
tags:
- AWS, terraform
---

I wrote a [Terraform module](https://github.com/saurabh-hirani/terraform-aws-elasticache-redis) which can be used 
for creating infra for running AZ attacks against different types of AWS Elasticache setups. This does 
not claim to be **the** best way to create AWS Elasticache Redis infra. Rather, it tries to help you 
get away with the minimal infra you need to try out different AWS Elasticache Redis setups. Read more 
on the need to create this module in the [repo](https://github.com/saurabh-hirani/terraform-aws-state-mgmt).
