---
layout: post
title: "PowerShell Style Guide"
subtitle: "Start with versioning"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# Style Guide

This is the first post in a series that will cover a style guide for PowerShell at my worklplace. Eventually, I will combine them into a large post that will be the full guide. 

# Code is Version Controlled

All of our code for reuse and dissemination should be in a version control system. We have chosen to use [Git](https://git-scm.com/about) on [AWS CodeCommit](https://aws.amazon.com/codecommit/). Even if you think the code will only be used as a one-off, you should version control it. Feel free to create a repository with your username as the name. You should not be providing these scripts to others for continuous use. 

Create a repository for specific related tasks. This doesn't mean all scripts related to Active Directory are in the same repository, but all scripts related to XST accounts, for instance, would be in a single repository. This helps to keep the commit history cleaner and more closely related to the code at hand. 

As you continue to modify your code, follow [semantic versioning](http://semver.org/). At a minimum, add tags to the repo for all major and minor revisions (i.e., when new features are introduced). 

Each repository should contain a README file. The standard is to use markdown, but if you prefer a different markup language and you get it cleared by your team, feel free. 

## The README file

See [Dan Bader's Blog](https://dbader.org/blog/write-a-great-readme-for-your-github-project) for more info on this.

The README should contain the following:
   * Name of project/tool
   * How to install it
   * Example usage
   * How to set up the dev environment
   * How to ship a change
   * Change log
   * License and author info

For our purposes, much of this will also be in comment based help (described below). Nevertheless, this will allow a user to see the important information without having to open the PowerShell itself. For our purposes, the most important pieces are shipping changes and the author info.

Normally, shipping changes is a way for the creator to explain to future contributors how they would add to the codebase in a consistent usable way. In our case, this should also include notes about the primary users of these scripts so that we can communicate to them that it has been updated.

We also want to include author info, including the position of the author at the time the script was conceived. The idea here is to provide contributors and users with an idea of whom they can contact if they need to understand the history of this script (NB: this does NOT mean that everyone should feel free to bypass the normal request/incident process if they have a feature request or find a bug). 