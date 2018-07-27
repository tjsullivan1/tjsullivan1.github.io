---
layout: post
title: "Evolving the Single Server EC2 Reference Architecture"
subtitle: "Terraform Locals and File Separation"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# Background

One of my colleagues recently introduced me to Terraform [locals](https://www.terraform.io/docs/configuration/locals.html) and explained how we could use these to simplify the deployment process by removing some of the hunting and searching in the main.tf file for variable information that was previously associated with several resources (e.g. tags).

# Separating the Files

Terraform allows a developer to separate the configuration into as many .tf or .tf.json documents as desired, so long as they are all within a specified directory. Originally, I had wanted to prevent file sprawl, but for reusability and simplicity, I have now broken files out as follows:
- additional.tf: This file is non-default terraform resources for this architecture, or resources that will change based on an architectural decision. An example of this is security group rules or route53 records (if a server is in a private subnet, I only have one DNS entry; if the server is in a public subnet, I have two DNS entries, one for each interface IP).
- backend.tf: The same as described in my other documents. This file specifies to Terraform where to store the state and how to lock the state file.
- main.tf: The static build of this architecture (e.g., the single server and single security group base resources) is put here. This file will not need to be modified by a developer if they copy from a different directory.
- user-data.sh: This file is separate from Terraform, but is used at instance deployment for configuration purposes.
- variables.tf: This file contains all of my variable data contained within a "locals" block. This is the only file that will need to be modified, and it will result in less chance of the reference architecture breaking.

# The Locals

A locals block looks something like this:
{% highlight bash %}
locals {
  Name                    = "reference-architecture"
  infoTaggingVersion      = "v2.0.0"
  itsProvisioner          = "Terraform"
  itsSecurityReviewDate   = "NEEDSTOBECONDUCTED"
  itsSecurityReviewTicket = "NEEDSTOBECONDUCTED"
}
{% endhighlight %}

We can then use them like so:

{% highlight bash %}
  Name                    = "${local.Name}"
  infoTaggingVersion      = "${local.infoTaggingVersion}"
  itsProvisioner          = "${local.itsProvisioner}"
  itsSecurityReviewDate   = "${local.itsSecurityReviewDate}"
  itsSecurityReviewTicket = "${local.itsSecurityReviewTicket}"
{% endhighlight %}

On the surface, this may not seem useful, but if our reference architecture contains multiple resources that all have a tag for infoTaggingVersion, instead of having to find and replace "v2.0.0" with "v2.0.1" throughout the main.tf (risking user error), we can now update it in the variables file and it will update throughout the document.

In addition, these will allow us to force more standards throughout the process. For instance, if all of my security groups should be "sg-systemname" and all EC2 instances should be "systemname", I can have one local `locals { Name = "systemname" }` that is then used in the security group (`Name = "sg-${local.Name}`") and the EC2 instance (`Name = "${local.Name}"`). No longer will I need to worry about copying and pasting with a weird space or misspelling when typing throughout a file.

Finally, this allows us to surface our standards in a reference architecture rather than a module file. If you take the previous security group/ec2 instance pairing scenario, we can do this same process by having "sg-" interpolated with the variable "${var.Name}", but when a developer uses the module to deploy a standard security group, our standards are hidden away from them if they have simply copied the file and they may never understand how or why their security group gets the name it ends up with.
