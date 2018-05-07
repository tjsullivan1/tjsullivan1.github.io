---
layout: post
title: "How I Build a New EC2 Instance"
subtitle: "Part 5 - Updating the Build"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# A Relatively Painless Process

The process of updating your Packer AMI is straightforward once you have your build down. First things first, I would follow the process detailed in [part 2](https://tjsullivan1.github.io/blog/2018/03/20/p2-get-ami-id) to see if there is a new version of your base AMI to be used. Next, you will want to do different things with the existing AMI based on which action you are taking. Ask yourself - am I changing the base config, the install script, or simply applying updates to the base AMI. 

If you are just running updates, `packer build build.json` will run updates and create a new updated AMI. I would recommend keeping the last version of the AMI, but you can feel free to deregister AMIs and remove snapshots for versions older than n-1. If you are changing the install script, edit your base install.sh command and then do as above, but purge any old AMIs. Oftentimes, the reason for changing the install is a config problem or change that you do not want to redeploy. Finally, if the base config changes (e.g., changing regions), you will want to edit the build.json file to make that change, once again removing any old AMIs to prevent the old configuration from being used.

Once you have the new AMI generated, copy the AMI ID and replace it in your main.tf file within your live repository. I would always recommend running `terraform plan` before `terraform apply` to make sure that nothing else in the environment has changed that is going to surprise you by destroying and recreating a resource. Please note that changing an AMI will DESTROY the existing instance and create a new instance. If you have any state tied to the system, you will need to change this process to account for copying the data.

Once you have run the apply command, you are all set with a newly generated instance running a new build.
