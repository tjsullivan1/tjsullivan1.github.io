---
layout: post
title: "The No-Prompt Switch"
subtitle: "Allowing script users to right click and 'Run with PowerShell'"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# The Challenge

Many of the admins that work with me don't work day-in, day-out in a shell. They like to be prompted for data that is needed by the script, and they don't like to have to open up a PowerShell just to run one command. This was easily accomplished by making a parameter mandatory (and avoiding those horrific read-hosts!). However, this introduces its own set of problems. If you run a command by right-clicking on the script, and the script throws an error, you have moments to attempt to capture an error. It is almost impossible, and there are few things more flabbergasting to a toolmaker than hearing "your script produced a lot of red text".

# The Solution 

To resolve this, I created the NoPrompt parameter. This is a switch parameter (meaning off is $false and on is $true) that I throw into the END block of every script. It looks like this:

{% highlight Powershell %}
    if ($NoPrompt) {
        Write-Verbose "You specified not be be prompted before exiting"
    } else {
       pause
    }
{% endhighlight %}

It is very simple. By default, we will use the ["pause" DOS command](https://technet.microsoft.com/en-us/library/bb490965.aspx), which will ask the admin running the tool to press enter to continue. This is very nifty because it will force the shell to remain open for them to capture any error text that is generated. If, however, we specify the -NoPrompt parameter (which cannot be done with a right-click) from a shell or scheduled task, the script will run as it would have without the parameter. This allows scheduling to be done without incident, while allowing for a fully functional tool to be used even if you are unfamiliar with PowerShell. 
