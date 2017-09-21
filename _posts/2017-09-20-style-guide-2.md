---
layout: post
title: "PowerShell Style Guide"
subtitle: "Comment Based Help"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# Style Guide

This is the second post in a series that will cover a style guide for PowerShell at my worklplace. Eventually, I will combine them into a large post that will be the full guide.

The previous post in this series is here:
   * [Start with versioning](https://tjsullivan1.github.io/blog/2017/07/15/start-style-guide)

# Add Comment Based Help to All Functions/Cmdlets

Here are several good explanations of comment based help:
   * [about_Comment_Based_Help](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_comment_based_help?view=powershell-5.1&viewFallbackFrom=powershell-Microsoft.PowerShell.Core)
   * [Don Jones' Comment your way to help](https://technet.microsoft.com/en-us/library/hh500719.aspx)

The core reason for this is to allow someone to run `get-help our-function`. However, this is meant to be a quick and easy to use style guide, not an explanation of making this work. The basics are that we want our help comments to be enclosed in a block comment (i.e. `<# #>`) to allow clean multi-line help.

## Minimum Usage

   * Every script or function should include the `.SYNOPSIS` keyword in the help. This should desribe the basic reason this script exists.
   * We also require `.PARAMETER` for each parameter that is included in the function definition. This should include an explanation of valid values for this parameter.
   * Finally, `.EXAMPLE` -- include a minimum of one example that includes the standard execution, an explanation of what that execution is doing, and the corresponding output.

## Additional Usage

   * `.DESCRIPTION` can be included if you want to add a detailed explanation of why the script exists and how it works. This should be added for scripts consumed by the help desk.
   * `.INPUTS` should be included if you script accepts pipeline input. Include a definition of what this input would be from. Do not add this for parameters.
   * `.OUTPUS` should be included if you script generates object outputs. Include a definition of how this output could be used. Do not add if your script isn't using `write-output` with an object. If your script is generating text to the screen, a csv, etc, it should be refactored or considered for refactoring.
   * `.NOTES` can be included if there is anything that you would think would be helpful for yourself or another individual to know. A past utilization of this field has included "FC" for "Future Consideration" to indicate functionality that we want to add later -- however, these notes would be better added to a work management system for tracking and prioritizatoin.

## Other Usage

There are many other potential comments to add to comment based help, but we are not currently using any of these at this time. If you have a use for an additional keyword, please let me know your intended usage and we can discuss adding it to this guide.

