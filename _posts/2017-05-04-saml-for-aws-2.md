---
layout: post
title: "SAML for AWS Part 2"
subtitle: "Configuring the CLI to use federated access"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# The Challenge

Yesterday, I walked through a configuration of AD FS & SAML2.0 for AWS Console access. This is all fine and dandy, until you realize that the real power comes from repeatable, automatable commands. Since a federated account doesn't actually exist as a user in AWS IAM, we don't have an access key or secret access key with which we can connect to the aws cli. The blog post below (and corresponding script) will get you 95% of the way there, but I needed to make a few tweaks for my environment.

Harvested from:
* [https://aws.amazon.com/blogs/security/how-to-implement-a-general-solution-for-federated-apicli-access-using-saml-2-0/](https://aws.amazon.com/blogs/security/how-to-implement-a-general-solution-for-federated-apicli-access-using-saml-2-0/)
* [https://s3.amazonaws.com/awsiammedia/public/sample/SAMLAPICLIADFS/samlapi_formauth_adfs3.py](https://s3.amazonaws.com/awsiammedia/public/sample/SAMLAPICLIADFS/samlapi_formauth_adfs3.py)

# The Solution 

First and foremost, my environment is mixed Windows and Linux. The script as it stood didn't work all the time with file paths. In addition, the script didn't do a great job of creating our credential file if this was our first time. So, I added some checking in there. Finally, I converted the script to python3 for future proofing. This required changing some modules around, but was fairly straightforward. The full script can be found [here](https://gist.github.com/tjsullivan1/200ffd6873b8e9d9497cd3d26f26897b). 

To get this to work, you will need Python 3 installed, and then you will need to install the following modules using pip: boto3, bs4, awscli, requests, configparser, lxml (your Python install may already have some of these, but mine did not have all of them). You will also need to modify the script for your IdP (you should only need to replace the placeholder adfs server name with the FQDN of your server). 

Below are some of the other modifications I made to the script: 
{% highlight python %} 
os_home = '~'
home = expanduser(os_home)
filepath = os.path.join(home, awsconfigfile)
filename = os.path.normpath(filepath)
{% endhighlight %}
This fixes pathing problems between Linux and Windows, it is a pretty standard usage of the os module in Python. Originally, I was expanding the userprofile environment variable on Windows with logic to determine OS, but this turned out to be superfluous. 

{% highlight python %} 
if not os.path.isdir(os.path.dirname(filename)):
    os.makedirs(os.path.dirname(filename))
{% endhighlight %}
I was having problems creating the file if it didn't exist when the .aws directory __also__ didn't exist. This statement will remedy that, by creating the directory.

{% highlight python %} 
mode = 'w+' if os.path.exists(filename) else 'w'
with open(filename, mode) as configfile:
    config.write(configfile)
{% endhighlight %}
Finally, we will create the file if it doesn't exist, but we will modify (not recreate) if it does. This prevents us from overwriting a non-SAML stored credential. 

# Usage Notes

Once you haver all of this in place, you can execute it by running "python aws_saml_auth.py". The script will prompt for username and password, and then it will display a selection screen for the user to pick what role they want to assume:
{% highlight python %} 
Please choose the role you would like to assume:
[ 0 ]:  arn:aws:iam::012345678901:role/S3FullAccess
[ 1 ]:  arn:aws:iam::123456789012:role/S3FullAccess
[ 2 ]:  arn:aws:iam::012345678901:role/ReadOnly
[ 3 ]:  arn:aws:iam::123456789012:role/Admin
[ 4 ]:  arn:aws:iam::012345678901:role/Admin
Selection: 
{% endhighlight %}
As you can see, we have roles appearing for our two accounts that were configured, and they map back to several AD groups that contain the AD account for the federated user authenticating. Once a selection has been made, if the script executes successfully, we output a helper message:
{% highlight python %} 
----------------------------------------------------------------
Your new access key pair has been stored in the AWS configuration file C:\Users\tjsullivan1\.aws\credentials under the saml profile.
Note that it will expire at 2017-05-02 22:54:10+00:00.
After this time, you may safely rerun this script to refresh your access key pair.
To use this credential, call the AWS CLI with the --profile option (e.g. aws --profile saml ec2 describe-instances).
----------------------------------------------------------------
{% endhighlight %}

This tells us how long we can use the credential for, and then how to use it (add "--profile saml" to all of our commands). You can change the AWS_DEFAULT_PROFILE to saml on your OS to circumvent that limitation, but it caused some problems for me when I tried to rerun the script once my SAML token expired. 

Some things that I plan to change/add in the future:
* Pull the friendly name of the account (e.g., 012345678901 is "Enterprise Test/Dev")
* Allow someone to create a custom profile name (e.g., RO-SAML or Admin-SAML) 
* Better error checking
* Duration settings to allow for extending/minimizing token duration