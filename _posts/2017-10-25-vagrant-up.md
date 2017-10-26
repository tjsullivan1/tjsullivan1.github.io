---
layout: post
title: "Vagrant Up!"
subtitle: "Standardizing a development environment"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# The Challenge

You may have seen some previous posts on my blog regarding configuring a git environment to use CodeCommit with SAML. There are several steps and depending on how someone runs certain commands, they have different behavior than I would see on my machine. For instance, if you install the git credential manager on Windows, the installation will succeed, but the AWS CLI and subsequent commands won't work with Git. In order to standardize, I built a Vagrant box (more on [Vagrant](https://vagrantup.com)) with all of the tools that we use (the Hashicorp toolchain, git, etc.)  as well as some scripts to quickly build a working environment for dev.

# The Solution

First, I tested this with Vagrant 2.0 and Virtual Box 5.1.30. You will need to install both of those in order to get this to work. Next, clone or download my Git repo `git clone https://github.com/tjsullivan1/its-vagrant-box`. From here, standard Vagrant commands will go a long way. To build your environment, open a command prompt or shell and change directories into the folder called 'its-vagrant-box'. Run `vagrant up`. This command will build the vagrant box by reading the Vagrantfile and will take several minutes to complete.

In the Vagrantfile, you will see that we are using a shell provisioner to execute the install.sh script. This script installs Python and dos2unix; updates pip to version 9; clones a gist that will download and install the Hashicorp tools; clones a git repo with the AWS auth python script; and uses pip to install the prerequisites for that script. Once that is done, you are ready to use the Vagrant box! Simply type `vagrant ssh` and you will be connected to your new guest OS.

The Vagrant box is running ubuntu/xenial64 instead of a more standard OS for my organization because I had issues with the shared directories. A Vagrant box, in theory,  has a directory '/vagrant' that is shared with the directory you created earlier. For instance, if you created the directory 'its-vagrant-box', you should be able to add files to it and see them in your '/vagrant' folder when you SSH into your Vagrant box. Nevertheless, I had trouble getting it to work between Virtual Box and CentOS 7 because of an error with either Virtual Box or Vagrant (my searches revealed the possibility of some vbox tools missing from the Vagrant box). Since the critical function of this box is to be able to quickly and easily load CodeCommit repos and run tools from both Linux and Windows, synced and shared folders are a necessity.

Once the box has finished building, run `vagrant ssh`. This will connect you to your box. Next, change directories to the /vagrant directory. Here, run `bash init_git_config.sh`. This script executes the aws_saml_auth script previously written about on this blog. Then, it properly configures git for CodeCommit in us-east-1 and us-east2. Next, it creates a directory called "Code" and changes directory into that directory. Finally, it clones a private (i.e., CodeCommit) repo with private tools into it. Currently, the only "tool" is an infrastructure admin repo cloner called "init_terraform_repo.sh", which will download all of the terraform repositories and modules for my org. I plan to add more tools eventually, and depending on their usefulness outside of my org, I might blog about them here.

Since we have all of the Hashicorp tools installed (as part of the Vagrant build), we can run packer, terraform, or other Hashicorp tool commands as we would if we were to have installed these on our own workstations. If you clone any repositories into the /vagrant/Code directory, you will be able to edit them with your favorite IDE or text editor from Windows or Linux. In addition, anytime you run `python3 /opt/tools/aws_saml_auth/aws_saml_auth.py`, you will reinitialize your credentials for CodeCommit and be able to run git commands as normal from within this Linux instance.

When you are done, exit the ssh session by typing `exit`, and then stop the Vagrant guest by typing `vagrant halt` or remove it entirely by typing `vagrant destroy` (nb: vagrant destroy requires you to rebuild from scratch).


# Up and Running

Here are the basic commands you can run to get up and running:

{% highlight bash %}
# Get the Vagrant Box
git clone https://github.com/tjsullivan1/its-vagrant-box
cd its-vagrant-box

# Build the box
vagrant up

# Connect to the box
vagrant ssh

# Create the initial git configuration
cd /vagrant
bash init_git_config.sh

# Reconnect to CodeCommit
python3 /opt/tools/aws_saml_auth/aws_saml_auth.py

# Stop the VM in Virtual Box
exit
vagrant halt

# Destroy the box
vagrant destroy

{% endhighlight %}

