---
layout: post
title: "SAML for AWS Part 3"
subtitle: "Configuring git to use federated access"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# The Challenge

The first two parts in this series have explained how I got federated access configured and working for both the AWS web console and the aws cli. One further feature that requires a small amount of tweaking is your git configuration (if you use AWS CodeCommit). Since we are federated users, and we don't have an access key and secret access key, we will need to add a credential helper to our git config in order to be able to use codecommit. In addition, we want to allow users to use other git services, so we will need to set the credential helper to a specific URL.

* For more info, see [this blog post from James Wing](https://jameswing.net/aws/using-codecommit-and-git-credentials.html)

# The Solution 

If you only use Git for AWS/work purposes, it is very simple to configure your git instance to work with your federated credentials. First, make sure you have the name of the profile that you created in your credentials file for your federated account. If you have been following along, for us, it is "saml". Next, we need to tell git how to connect to CodeCommit. This is done with the following two shell commands:

{% highlight bash %} 
git config --global credential.helper '!aws codecommit credential-helper $@ --profile=saml'
git config --global credential.UseHttpPath true
{% endhighlight %} 

These are telling git to use the aws codecommit command with your saml profile and to use an http connection (not SSH) to the repository. You can then do your standard git commands to clone, pull, push, etc. 

However, if we use git for personal use, or on a team not utilizing AWS CodeCommit, we will need to do domain specific helpers. In order to do that, we run the following command:

{% highlight bash %} 
git config --global credential.https://git-codecommit.us-east-1.amazonaws.com.helper '!aws --profile saml codecommit credential-helper $@'
{% endhighlight %} 

Rerun that command for any repositories that you might have in the Ohio, Oregon, or Ireland regions with their specific region URL. You will need to run a corresponding command for any other git services you use (e.g., https://github.com). 