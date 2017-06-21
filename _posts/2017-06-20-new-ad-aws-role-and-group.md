---
layout: post
title: "Create a New AWS Role and AD Group with PowerShell"
subtitle: "Quickly create a new role with a matched AD group for SAML"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# The Challenge
With our AD, AD FS, and AWS setup previously described, I discussed configuring roles and AD groups. However, doing so manually is error prone. The following scripts automate the steps and prevent a lot of copy-pasting, especially when multiples are being created at one time.

I am investigating using Hashicorp's Terraform for managing IAM roles to prevent the manual step of creating these roles (and their corresponding policies). Once I have my process standardized, I will post a solution. 

# The Solution
The full code can be found in the following gists: [New-ADAWSRoleGroup.ps1](https://gist.github.com/tjsullivan1/8aaf4cf4d811a63f126c578a687b192d) and [New-AWSIAMRole.ps1](https://gist.github.com/tjsullivan1/2191e1b037172e3f02e466a533fe4f66).

## New-AWSIAMRole 
NB: You should have a pre-existing policy in AWS before running the cmdlet. 

This cmdlet accepts two parameters. First, we specify what we want the role to be called in AWS. This will match the end of the AD group name that we created earlier. The second is the policy name, this should be in AWS already. It can be an AWS-managed or a customer-managed policy. 

From here, we use the `Get-EC2SecurityGroup` cmdlet (with our credentials loaded into the shell), by returning the default group, we are able to return our account ID without needing any additional info. This is helpful for not having to copy paste later.

This is then used to build our account arn. Next, we create the trust policy (the piece that specifies that SAML can be used to assume this role). We use a here string for the JSON of the trust policy, and then use the string replace method to replace our ARN.

Then, we create the role with this role assumption policy, and finally, we register the actual permission policy to make that role effective. 

## New-ADAWSRoleGroup
This script is quick to explain and use. To run it, you specify a role name and your account number (you could build in a similar get- command like the previous cmdlet, but then a new dependency is introduced). We then build a name variable that matches our SAML query string in our claim rules (AWS-AccountNumber-Role). Then, with the New-ADGroup, we create an AD group with a basic description and display name. 

From here, you can add or remove users as usual in AD. From there, once a user in the group connects with SAML to the GUI or CLI, they will login with that role (if they are only in one AD group), or they will see a prompt to select which role they want to assume (it will appear just as the Role_Name, not with the account number, arn, or AWS in the name). 