---
layout: post
title: "How I Build a New EC2 Instance"
subtitle: "Part 4 - Terraform Apply"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# Background

[Terraform](https://www.terraform.io/) is an infrastructure as code tool, allowing you to codify your infrastructure across multiple providers (e.g., AWS, Azure) with one standard workflow. I use Terraform for much more than EC2, but this post will only focus on how I would deploy a single EC2 instance. You should have a basic understanding of Terraform if you want to understand the contents of this post.

# Process for Single Instance

## High-Level Directory Structure and File Breakdown

I have a repository in AWS CodeCommit that stores all of my "live" terraform files. I won't claim to have the best setup, but based off of what I was reading around the time I started this a year or so ago, this seemed acceptable:
{% highlight bash %}
tf-live
- prod
-- iam
-- remote-state
-- us-east-1
--- networking
---- vpc
--- s3
--- services
---- my-new-ec2-instance
----- README.md
----- user-data.sh
----- main.tf
----- backend.tf
- test
-- iam
-- remote-state
-- us-east-1
--- networking
---- vpc
--- s3
--- services
---- my-new-ec2-instance
----- README.md
----- user-data.sh
----- main.tf
----- backend.tf
- dev
-- iam
-- remote-state
-- us-east-1
--- networking
---- vpc
--- s3
--- services
---- my-new-ec2-instance
----- README.md
----- user-data.sh
----- main.tf
----- backend.tf
{% endhighlight %}

As you can see, I break down by environment, then by location (non-regional services are at the top level), then by region, then networking, s3, or service implementations. Each service gets its own folder, which will contain one file that invokes one or more modules to build the infrastructure necessary for that region.

I also have separate repositories for each of my modules (because CodeCommit doesn't work like GitHub for referential linking to paths within the repo). These look something like this:
{% highlight bash %}
tf-single-server
- CHANGELOG.md
- main.tf
- outputs.tf
- README.md
- vars.tf
{% endhighlight %}

I'll break down each of the files in turn, starting with those in the module directory (in this case, tf-single-server).

## tf-single-server

### Non-Terraform Files

Each repository I build has two non-terraform files. First, is a changelog (CHANGELOG.md) that attempts to follow the practices laid out by [keep a changelog](https://keepachangelog.com/en/1.0.0/). This is a good addition to the git log that is added when I or one of my colleagues changes the repository because it takes more thought to add to. Second, I have a readme (README.md) that follows the example from [Dan Bader](https://dbader.org/blog/write-a-great-readme-for-your-github-project). These two documents are for other admins to be able to figure out what this particular module is for and to understand the evolution of the project.

### vars.tf

The [vars](https://gist.github.com/tjsullivan1/5ed67d4043c23b10eec481767e35cb4d#file-vars-tf) file is the easiest to understand. It is a file that contains variable definitions, which appear like so:

{% highlight bash %}
variable "infoTaggingVersion" {
  # Example Value: v#.#.#
  description = "Describes the version of this document to reference to understand tags."
  default     = "v2.0.0"
}
{% endhighlight %}

The variable itself is within quotes. The description will display if you are interacting with terraform from the command line and haven't added a value. The default value is used if no value is provided. If there is no default, and no value is provided, terraform will prompt (or error depending on the method of invocation).

### outputs.tf

The [outputs](https://gist.github.com/tjsullivan1/5ed67d4043c23b10eec481767e35cb4d#file-outputs-tf) file is the next easiest to understand. This defines queryable fields that will be available for reference after terraform creates the resource. These values can be reference by running `terraform output <resource.output_name>` or using `<module.output_name>` in a terraform file that has built the resource as part of a module.

### main.tf

The [main](https://gist.github.com/tjsullivan1/5ed67d4043c23b10eec481767e35cb4d#file-main-tf) file is probably the hardest to understand. The long and the short of it is that I use two data sources and create two resources as part of the build. The first data source that we return is the vpc that matches the "VPC Level" defined in the vars.tf file. For me, this is either a public VPC or a campus VPC, with the difference being campus resources are only accessible via a point-to-point VPN with a device on my local network. We then pass that value to the second data source to return a list of subnets in that VPC that match "subnet type" (i.e., either public or private). Next, we pass that to a random_shuffle resource and get the ID of one of those subnets. Finally, we get to the meat of this module.

The main portion of the module is dedicated to creating an aws_instance resource (i.e., an EC2 instance). As you can see, every single option references a variable from the vars.tf file. The majority of the options that we use are tags, which conform to a standard tagging policy.

## tf-live

### README.md

This readme file is to clarify any manual steps that need to be taken with this instance. An example would be a server that has a non-scriptable config step (i.e., entering a password in a popup). We would then put a note in here so any terraform user would know that running `terraform apply` would not be sufficient.

### user-data.sh

This file must exist with my terraform module, even if it is blank. The purpose of this file is to be able to execute a series of commands when the instance is launched. More information can be found in the [EC2 User Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html#user-data-shell-scripts).

### backend.tf

This guide is assuming that you have already configured your terraform backend. This file specifies to Terraform where to store its state and how to lock the state file if we are collaborating. An example:
{% highlight bash %}
provider "aws" {
  region = "us-east-2"
}

terraform {
  backend "s3" {
    bucket         = "remote-state-development"
    key            = "us-east-2/services/bastion/terraform.tfstate"
    region         = "us-east-2"
    dynamodb_table = "state-locking-development"
  }
}

{% endhighlight %}

The way this file works is beyond the scope of this post, but be sure to change the `key` entry to match the path to your system (i.e., don't leave bastion in there) or you will have a bad day when you overwrite the state of another service.

### main.tf

This file is the culmination of everything else. It will look something like this at its most basic:

{% highlight bash %}
module "bastion_server" {
  source = "git::https://git-codecommit.us-east-1.amazonaws.com/v1/repos/tf-single-server?ref=v0.9.1"

  # Inputs
  ami                      = "ami-##############"
  key_name                 = "testing"
  assoc_public_ip          = "False"
  subnet_type              = "Public"
  security_group_id        = "${module.security_group.sec_group_id}"
  vpc_level                = "Campus"
  instance_type            = "t2.micro"
  instance_role            = ""

  # Tags
  # Mandatory
  ## Has Default
  ###infoTaggingVersion =
  ###itsProvisioner =
  ###itsSecurityReviewDate =
  ###itsSecurityReviewTicket =
  ###CA001 =

  ## No Default, Needs Value
  CostCenter            = "#####"
  itsDataClassification = "Yellow"
  itsServiceOwner       = "me@me.com"
  Creator               = "me@me.com"
  Department            = "IT"
  Environment           = "DEV"
  Name                  = "bastion"
  Owner                 = "me@me.com"
  Project               = "Cloud Migration"
  Purpose               = "Testing a base build"
  Service               = "Amazon Web Services"

  # Optional
  ###itsBackupSchedule =
  ###itsBuildID =
  ###itsCmdbMapping =
  ###itsOperationalHours =
  ###Application =
  ###Company =
  ###CA002 =
  ###CA003 =
  ###Role =
  ###Status =
  ###Use =
}
{% endhighlight %}

What does this do? First, we are creating a resource (or resources) from a module. This is what `module "bastion_server" {}` is doing. The module is called "bastion_server", you can feel free to change that to whatever you want, but anywhere else that you are going to reference it you will need to use that name. For instance, after this system is deployed, if I wanted to get its public IP (defined in outputs.tf), I would reference `${module.bastion_server.public_ip}`. If I changed the name to "web_server", I would do the same with `${module.web_server.public_ip}`.

Next, we define the source for this module `source = "git::https://git-codecommit.us-east-1.amazonaws.com/v1/repos/tf-single-server?ref=v0.9.1"`. That line tells Terraform that the source is a git repo in CodeCommit us-east-1 called tf-single-server and I want to get the commit tagged 'v0.9.1'. It is critical that you start to tag your modules and specify those in your live builds. If you do not, when someone updates the module code your whole build will change with it when you next run `terraform apply`.

Next, we have a series of inputs. We generated the AMI ID with our earlier [Packer build](https://tjsullivan1.github.io/blog/2018/03/20/p3-packer-build). The key_name is the SSH key that you have built for this region without the .pem extension. `assoc_public_ip` will grab an Amazon Public IP address if set to true. __NOTE:__ this will not grab an _Elastic_ IP, so if you need a static IP you will need to specify this separately. Then, we tell it whether we want a public or private subnet. This is followed by the security group that should be attached to this instance. You can see here that I am referencing a module called "security_group" to get its ID for use here. You could specify a pre-existing security group by ID in this field instead.

Next, we tell it which VPC we want to deploy to by "VPC Level" (an internal naming mechanism explained above). Next, we tell it what instance type this instance should be - this will match the AWS standard of "instance family.instance size". Then, we have `instance_role`. This value can be left null, but if you need the instance to have permissions to other resources in AWS, put the friendly name for the IAM role here.

Finally, we come to tags. I have this broken down into three sections for my organization. First, we have mandatory tags that have a default value (for instance, infoTaggingVersion is a tag with the key "info:TaggingVersion" and the value "v2.0.0" defined in our module's vars.tf file). Next, we have mandatory tags that have no default value. If these aren't added, terraform will error. Finally, we have a series of optional tags. These tags have a default value of null, but we are able to add them if we want here.

## Building with Terraform

Once your file structure is set up, the repository committed, etc. you can create your new instance by doing the following in your tf-live/dev/us-east-2/services/service_name/ directory:
{% highlight bash %}
# Connecting the terraform backend and downloading the module
terraform init

# Build your execution plan, make sure you aren't going to break anything
terraform plan -out `date +%Y%m%d-%H%M`.plan

# Create your resources
terraform apply <date_command_output>.plan

{% endhighlight %}

After you have applied, you will have an EC2 instance deployed! Other things that I will often do in the main.tf file here is create the security group, attach any security group roles, or maybe add a Route53 record:
{% highlight bash %}
resource "aws_route53_record" "main" {
  zone_id = "<ZONE_ID>"
  name = "bastion.my-domain.com"
  type = "A"
  ttl = "300"
  records = ["${module.camp_server.private_ip}"]
}
{% endhighlight %}

# References

For more self-paced learning on Terraform, I recommended [The Terraform Book](https://terraformbook.com/) by James Turnbull or [Terraform, Up and Running](https://www.terraformupandrunning.com/) by Yevgeniy Brikman.
