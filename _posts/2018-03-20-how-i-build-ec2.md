---
layout: post
title: "How I Build a New EC2 Instance"
subtitle: "Part 1 - Overview"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# How I Build a New EC2 Instance - Part 1 - Overview

In the next series of posts, I am going to walk through my thought process between the deployment of services using Amazon EC2 with automated tooling. This page may be updated after its initial publication date if I add more to the series. As it sits now, this will be a five-part series (including the overview):
1. Overview
2. Getting Your AMI ID
3. Packer Build
4. Terraform Deployment and Configuration Options
5. Updating the Build

## Getting Your AMI ID

This section covers using a script that I have written to return the AMI ID based on criteria that I have selected. This may not be the best way, but it is my way, and I haven't come across a better one.

## Packer Build

This section covers building a master AMI for you to further build on, as well as an example service build.

## Terraform Deployment and Configuration Options

This section covers how my "single-server" Terraform module works as well as some additional configurations that I prefer when I deploy a new service using EC2.

## Updating the Build

This section covers recreating your Packer AMI, cleaning up the old versions, and redeploying with Terraform. Some of the Terraform options are re-covered to discuss how they do or do not affect the existing instances.

