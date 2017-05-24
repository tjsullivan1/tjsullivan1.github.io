---
layout: post
title: "Hubris and Idempotence"
subtitle: "...or just because they're wrong, it doesn't mean you're right!"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# The Challenge

There were two underlying issues, one technical and one social. First the technical: we had several users whose mailboxes disappeared. Normally, I am not one to believe in "magic", but when you can't find a tombstone, an inactive mailbox, or a soft-deleted mailbox, it sure doesn't sound like something done via a computer! 

The second issue was social. One of my colleagues was working with Microsoft to rectify this situation and he discovered that adding an Azure Active Directory Premium license to the user reconnected the missing mailbox. Now, I have never seen anything like that, and the Microsoft support rep told my colleague that it was because user-writeback doesn't work if you don't have an Azure AD Premium license. That is not correct to my understanding - user-writeback has gone through various iterations of being enabled, disabled, and partially in place (for Exchange attributes, no less). Never in my recollection has that revolved around a specific (paid!) SKU in Office 365. The Exchange Online licenses should be sufficient to synchronize those attributes (in fact, without that functionality, syncing would not work). I told the support rep that, and he quickly backtracked and then said that we must have had the Exchange license removed for more than 30 days. We knew from Azure AD logs and the user report that the user had been using Exchange Online yesterday, so I told him that was not the case either. 

Nevertheless, he persisted and said one of our admins must have deleted the Exchange mailbox. I am unaware of a way to delete a mailbox in such a way that there is no trace and yet have it be recoverable. If we had deleted the mailbox so thoroughly that searching inactive and soft-deleted mailboxes didn't find it, we shouldn't be able to restore anything! In addition, we have auditing turned on, and we are able to see dozens of other actions taken by our admins and automation account. Why can't we see this? (The last is still unsolved.)

Here is what we learned. We DID remove the Exchange Online license, through a script that was meant to be idempotent if run against an active user. This script sets licenses correctly for a complicated licensing setup that we have at our organization, and running it ten thousand times should result in the same behavior and end state. This user never changed roles in our identity management, so what happened?

# The Solution

It turns out that I had made a fatal flaw while writing my script. Consider this one of a thousand examples why I need to become more like a developer! I assumed that the subsystems would be available when I ran my scripts. 

In the code, we look at the AD user on-prem for the distinguished name and base our script off of the OU location. However, I didn't consider what happens if AD is unreachable. In this case, I looked for a distinguished name containing "CN=Users" (or two other OUs indicative of an active user). However, if you run $dn = get-aduser username -property distinguishedname | select -expand distinguishedname, and the command errors, $dn will be null. That means that any pattern matching for a value will result in a boolean false. If the if statement was false, the else statement removed the licenses to prevent us from spending money on unnecessary licenses. 

In order to resolve this, I added a check in an else if statement -- (!($dn)). This statement will catch an error connecting to AD (or just a missing user) by writing a warning and exiting the current loop without running any of our licensing. In a future post, I may explain the whole script further.

All in all, despite the technical ramifications, this was a poignant reminder that I'm not always as good as I might assume. It is good to be reminded that, at the end of the day, someone else being wrong doesn't make you right. 