---
layout: post
title: "Getting Membership of a Unified Group"
subtitle: "Slightly different than Security or Distribution Groups"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# The Challenge

In the spirit of unnecessary complication, Office 365 groups (i.e., unified groups) do not match standard cmdlet convention for other groups. With an AD security group, you can run the 'get-adgroupmember' cmdlet. With an Exchange DG, you can run the 'get-distributiongroupmember' cmdlet. Now, one wouldn't be hard pressed to imagine that the corresponding group cmdlet for unified groups would be 'get-unifiedgroupmember', but one would be wrong. 

# The Solution 

In the spirit of fairness, Microsoft's change, does make sense. A unified group has several different possible lists of users (owners, members, subscribers). In the case that we want to validate someone is a member (much like we would with a DG), we would run the 'Get-UnifiedGroupLinks' cmdlet. The full context looks like this:

{% highlight Powershell %}
Get-UnifiedGroupLinks -Identity <GROUP>@<TENANT> -LinkType members
{% endhighlight %}

A short one this week, but this was a useful thing I learned this week. 