---
layout: post
title: "How I Build a New EC2 Instance"
subtitle: "Part 3 - Packer Build"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# Background

[Packer](https://www.packer.io/) is a image automation tool, allowing you to build images for many different hypervisors. At this time, I only use it for Amazon Machine Images.

# Process for Base AMI

## High-Level Directory Structure and File Breakdown

I have a repository in AWS CodeCommit that stores all of my packer build files. Each build gets its own folder, the structure looks like this:
{% highlight bash %}
eieio-packer
- base_build_amazon
-- build.json
-- install.sh
- base_build_rhel
-- build.json
-- install.sh
- base_build_ubuntu
-- build.json
-- install.sh
{% endhighlight %}

Initially, I began naming the json build files "*application*.json" (where application was substituted for the app or purpose for this image), but one of my colleagues pointed out that for consistent commands, if we changed the build file name to build.json, `packer build build.json` could be used in any directory. Each directory also contains an install.sh script for basic commands. These files are broken down below.

## build.json

The base build file for packer has a specific format:

{% highlight js %}
{
  "variables": {
    "aws_access_key": "",
    "aws_secret_key": ""
  },
  "builders": [{
    "type": "amazon-ebs",
    "access_key": "{{user `aws_access_key`}}",
    "secret_key": "{{user `aws_secret_key`}}",
    "region": "<<REGION>>",
    "source_ami": "<<AMIID>>",
    "instance_type": "t2.micro",
    "ssh_username": "ec2-user",
    "ami_name": "Base Amazon Linux -- {{isotime \"2006-01-02T1504Z\"}}  -- built by Packer"
  }],
  "provisioners": [{
    "type": "shell",
    "script": "install.sh"
  }]
}
{% endhighlight %}

The variables are unused here, but if you aren't using a federated access user, you can insert static values in here. However, if you are federated, packer will pull your access key and secret key from the default profile.

In the builders section, we specify that we want to build an ebs backed instance. <<REGION>> and <<AMIID>> shouuld be specified by you as you prepare to deploy these. You can get the AMI ID with the [get_recent_ami script]() and you should know what region you want to use. The ssh_username will need to match tthe username of the default AMI user for your OS. The ami_name field specifies the value for the "AMI Name" field in the AWS Console. The isotime value is interpolated to become the current time in an ISO 8601 compliant timestamp. We then append -- built by packer so that we know this was built via automation.

Finally, we have the provisioners section. This part can be fairly powerful, but for a base build we only use a shell provisioner, telling it to execute the script "install.sh".

## install.sh

{% highlight bash %}
#!/bin/sh -x

# Apply updates
sudo yum update -y

# Build user for TJS
sudo adduser tjs -u 9999999
sudo mkdir -p /home/tjs/.ssh
sudo chmod 700 /home/tjs/.ssh
sudo touch /home/tjs/.ssh/authorized_keys
sudo chmod 600 /home/tjs/.ssh/authorized_keys
echo "<<RSA_KEY>" | sudo tee -a /home/tjs/.ssh/authorized_keys
sudo chown -R tjs:tjs /home/tjs

# Change our timezone to central US
sudo timedatectl set-timezone America/Chicago

# Don't require passwords for sudo
sudo sed -i "s/^# %wheel/%wheel/" /etc/sudoers

# Add our users to sudoers
sudo usermod -aG wheel tjs
{% endhighlight %}

The install script is straightforward. First, we display all outpt ot the display with #!/bin/sh -x (this helps with troubleshooting output errors during the build process). Next, we apply all updates to this machine. Next, we add a user with a prebuilt public ssh key (this will generate a 4096bit key: `ssh-keygen -t rsa -b 4096 -f ~/.ssh/tjs.key -C "Tim Sullivan's Key"`; two files are generated, you need the contents of the .pub file for the echo statement and the other you will need to login). Repeat this step for any user who will need direct access to the base build and any builds subsequently made from the base build. This keeps us from needing to login as ec2-user, allowing for [non-repudiation](https://en.wikipedia.org/wiki/Non-repudiation) and prevents sharing a private key. Next, my team is based in the Central US time zone, so we set the timezone to central. Then, I enable sudo without password for hte wheel group by changing the sudoers config. Finally, I add my newly created user to wheel to grant full admin permissions. **Do NOT copy the last line for all users - only for adminsitrative users.**

## Building the AMI

Once you are satisfied with your config, ensure you are authenticated to the proper AWS account and build a new AMI with the command `packer build build.json`. This will take several minutes, but when it completes it will output a new AMI ID.

# Process for Application-specific AMIs built from Master AMI

## build.json

The general build.json file stays relatively the same, with one key exception. Copy the master AMI ID from the previous packer output and replace the source_ami value with that. You will also want to change the ami_name value to properly indicate the purpose of this AMI, but leave the isotime and --built by Packer values, since we will still want those to differentiate from AMIs built in another way.

Sometimes, you will want additional provisioners as well. For instance, if I had the Oracle instant client rpm and I wanted to install it, I would copy that file into the folder containing my build file and install script and add this to the provisioners:
{% highlight js %}
{
    "type": "file",
    "source": "oracle-instantclient12.2-devel-12.2.0.1.0-1.x86_64.rpm",
    "destination": "/tmp/oracle-instantclient12.2-devel-12.2.0.1.0-1.x86_64.rpm"
},{
    "type": "shell",
    "script": "install.sh"
  }
{% endhighlight %}

This will copy the local file to the /tmp directory on the AMI. From there, we can do something with that file in the install.sh script.

## install.sh

This install script can be used for many additional things. Add additional packages needed for your app to run; install your app; add addional users; configure cron; the list is endless! A prime example was mentioned in the previous section. After we have copied the file to /tmp, we can install it with `sudo yum -y install /tmp/oracle-instantclient12.2-devel-12.2.0.1.0-1.x86_64.rpm` in our install script.

## Building the AMI

As with the master AMI, once you are satisfied with your config, ensure you are authenticated to the proper AWS account and build a new AMI with the command `packer build build.json`. This will take several minutes, but when it completes it will output a new AMI ID. You will use this AMI when you deploy the new system using Terraform.

# References

For more self-paced learning on Packer, I recommended [The Packer Book](https://packerbook.com/) by James Turnbull.
