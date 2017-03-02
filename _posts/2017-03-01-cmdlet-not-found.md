---
layout: post
title: "Troubleshooting PowerShell Errors"
subtitle: "Not recognized as the name of a cmdlet"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# The initial message
>Hey Tim,
>Not an emergency, but I had a chance to try the –IsCloud for a secondary mailbox.  It didn’t work so I just did it the old way and will let migrate. 
>Here’s the output I got on the screen:

{% highlight Powershell %}
PS C:\Users\colleague> C:\Automate\Exchange\secondaryMailboxCreation\new-ustsecondarymailbox.ps1 -IsCloud
cmdlet new-ustsecondarymailbox.ps1 at command pipeline position 1
Supply values for the following parameters:
(Type !? for Help.)
Name: My Secondary Mailbox
Alias: mysecondarymailbox
Members[0]: USER1
Members[1]: USER2
Members[2]: USER3
Members[3]:
Owner: USER3
C:\Automate\Exchange\ConnectToO365\Connect-ToO365.ps1 : The term
'C:\Automate\Exchange\ConnectToO365\Connect-ToO365.ps1' is not recognized as the name of a cmdlet, function, script
file, or operable program. Check the spelling of the name, or if a path was included, verify that the path is correct
and try again.
At C:\Automate\Exchange\secondaryMailboxCreation\new-ustsecondarymailbox.ps1:60 char:9
+         C:\Automate\Exchange\ConnectToO365\Connect-ToO365.ps1
+         ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (C:\Automate\Exc...nect-ToO365.ps1:String) [], CommandNotFoundException
    + FullyQualifiedErrorId : CommandNotFoundException
New-Mailbox : The term 'New-Mailbox' is not recognized as the name of a cmdlet, function, script file, or operable
program. Check the spelling of the name, or if a path was included, verify that the path is correct and try again.
At C:\Automate\Exchange\secondaryMailboxCreation\new-ustsecondarymailbox.ps1:95 char:5
+     New-Mailbox -Name $Name -Alias $Alias -OrganizationalUnit "sullivanenterprises.org/Spec ...
+     ~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (New-Mailbox:String) [], CommandNotFoundException
    + FullyQualifiedErrorId : CommandNotFoundException
Add-MailboxPermission : The term 'Add-MailboxPermission' is not recognized as the name of a cmdlet, function, script
file, or operable program. Check the spelling of the name, or if a path was included, verify that the path is correct
and try again.
At C:\Automate\Exchange\secondaryMailboxCreation\new-ustsecondarymailbox.ps1:100 char:5
+     Add-MailboxPermission -Identity $identity -User UST\$group_name -AccessRight ...
+     ~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (Add-MailboxPermission:String) [], CommandNotFoundException
    + FullyQualifiedErrorId : CommandNotFoundException
{% endhighlight %}

The errors continue, but are not critical to this discussion. 

# Breaking it down

PowerShell errors are sequential, so you should always focus on the first error message you see, fix that, and then retry. Move forward one error at a time. 

So, to do that:

{% highlight PowerShell %}
C:\Automate\Exchange\ConnectToO365\Connect-ToO365.ps1 : The term
'C:\Automate\Exchange\ConnectToO365\Connect-ToO365.ps1' is not recognized as the name of a cmdlet, function, script
file, or operable program. Check the spelling of the name, or if a path was included, verify that the path is correct
and try again.
At C:\Automate\Exchange\secondaryMailboxCreation\new-ustsecondarymailbox.ps1:60 char:9
+         C:\Automate\Exchange\ConnectToO365\Connect-ToO365.ps1
+         ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (C:\Automate\Exc...nect-ToO365.ps1:String) [], CommandNotFoundException
    + FullyQualifiedErrorId : CommandNotFoundException
{% endhighlight %}

In this case, our first exception is related to a cmdlet not being found. The text {% highlight PowerShell %}'At C:\Automate\Exchange\secondaryMailboxCreation\new-ustsecondarymailbox.ps1:60 char:9'{% endhighlight %} tells you where to look in the script to find the exception message. The text {% highlight PowerShell %}'C:\Automate\Exchange\ConnectToO365\Connect-ToO365.ps1'{% endhighlight %} tells you what command generated this message. In this case, let's look at that in the code...

{% highlight PowerShell %}
 BEGIN {
    Import-Module ActiveDirectory
    if ($IsCloud) {
        C:\Automate\Exchange\ConnectToO365\Connect-ToO365.ps1
    } else {
        # This session business rectifies a watson dump error...
        $Session = New-PSSession -ConfigurationName Microsoft.Exchange -ConnectionUri $ConnectionURI -Authentication Kerberos -Name NewSecondaryMailbox
        try {
            Import-PSSession $Session -AllowClobber -ErrorAction Stop -Verbose:$False # Allow clobber makes it so that the script doesn't attempt to use internal exchange cmdlets, which will fail
        } catch {
            Write-Warning "Session import failed. This script is not going to run properly."
            exit # quit the script before something bad happens.
        }
    }
}
{% endhighlight %}

We can see that this is probably a script that hasn't been edited in a while by the dates. The offending line is: {% highlight PowerShell %}C:\Automate\Exchange\ConnectToO365\Connect-ToO365.ps1
{% endhighlight %}
Since the exception is that the cmdlet couldn't be found, let's look at that directory:

{% highlight PowerShell %}
PS C:\Users\tjsullivan2> gci C:\Automate\Exchange\ConnectToO365\
gci : Cannot find path 'C:\Automate\Exchange\ConnectToO365\' because it does not exist.
At line:1 char:1
+ gci C:\Automate\Exchange\ConnectToO365\
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (C:\Automate\Exchange\ConnectToO365\:String) [Get-ChildItem], ItemNotFou
   ndException
    + FullyQualifiedErrorId : PathNotFound,Microsoft.PowerShell.Commands.GetChildItemCommand
{% endhighlight %}

Well, there you have it. We don't have a directory that contains the file. What happened? Likely, we modified the structure of all of our scripts and didn't have note of all of the scripts dependencies. In this case, we can resolve this error by updating the code to point to the new code location: C:\Automate\Includes\ConnectToO365\Connect-ToO365.ps1

Going to the next error in line, you can see that it is the same general error as before:

{% highlight PowerShell %}
New-Mailbox : The term 'New-Mailbox' is not recognized as the name of a cmdlet, function, script file, or operable
program. Check the spelling of the name, or if a path was included, verify that the path is correct and try again.
At C:\Automate\Exchange\secondaryMailboxCreation\new-ustsecondarymailbox.ps1:95 char:5
+     New-Mailbox -Name $Name -Alias $Alias -OrganizationalUnit "sullivanenterprises.org/Spec ...
+     ~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (New-Mailbox:String) [], CommandNotFoundException
    + FullyQualifiedErrorId : CommandNotFoundException
{% endhighlight %}

In this case, the error makes sense. The New-Mailbox cmdlet is loaded as part of the Connect-ToO365 script. The remaining errors are similar cmdlets that failed to load when we didn't create the connection to Office 365 or are errors because they are trying to modify a mailbox that failed to create.
