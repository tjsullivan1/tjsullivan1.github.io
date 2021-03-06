---
layout: post
title: "End User Quick Start for AWS & SAML"
subtitle:
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# Up and Running
I am going to assume that your AWS SAML configuration is complete, and this will be much more bulleted than other posts. You also must be in at least one AD group assigned to a AWS role

__NOTA BENE:__ this content is still valid, but I am now recommending the usage of a tool like my [Vagrant box](https://tjsullivan1.github.io/blog/2017/10/25/vagrant-up) for a standardized development experience.

## Console
* To log into the console, go to your ADFS Server's IDP initiatied sign on page: [https://\<ADFS_SERVER\>/adfs/ls/IdpInitiatedSignOn.aspx?loginToRp=urn:amazon:webservices](https://\<ADFS_SERVER\>/adfs/ls/IdpInitiatedSignOn.aspx?loginToRp=urn:amazon:webservices)

## CLI 
* To log into the cli, install Python 3
   * Depending on your OS, this may involve downloading the package from [python.org](https://www.python.org/downloads/) or installing via package manager
* Make sure that pip is installed for your version of Python
* Run (as an admin) `pip install boto3 bs4 awscli requests configparser lxml plac`
* Download the [auth script](ttps://gist.github.com/tjsullivan1/200ffd6873b8e9d9497cd3d26f26897b)
* Replace line 37 with the URL of your ADFS Server.
* Run `python aws_saml_auth.py` to login with your AD credentials (upn/password)
* Run AWS commands with the following syntax `aws --profile=saml \<cmd\>`

## Git
* Configure CLI first
* Install git
* From a shell, run:
   * `git config --global credential.https://git-codecommit.us-east-1.amazonaws.com.helper '!aws --profile saml codecommit credential-helper $@'`
   * `git config --global credential.https://git-codecommit.us-east-1.amazonaws.com.UseHttpPath true`
* Connect with the `python aws_saml_auth.py`
* Run your git commands as normal
