---
layout: post
title: "Friendly Identity Parameter"
subtitle: "Toolmaking for Office 365/AD Ease of Use"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# The Challenge

The office 365 cmdlet set does not typically include the -Identity parameter familiar to many AD admins and instead uses the -UserPrincipalName. In practice, this makes sense from an underlying AD vs. AAD perspective, but it makes it less friendly to provide tools to colleagues that might not know or might not care about the intricacies of synced directories. One of the ways my team overcame this was by adding the identity parameter and fixing it in the script if the script was going to modify O365 accounts. However, this introduced its own problems - how should someone know if the parameter is going to accept a user principal name or a samaccountname? Eventually, I plan to make this a more robust module that will allow for all sorts of different identity information to be used to find the user account and pass the correct identifier to later sections of complex scripts.  

# The Solution 

While I plan to expand this further, the quick solution that I developed for this is below:

{% highlight Powershell %}
    foreach ($user in $Identity) 
    {
        if ($user -notlike "*@sullivanenterprises.org") {
            $user = $user + "@sullivanenterprises.org"
        }

        try {
            Get-MsolUser -UserPrincipalName $user -ErrorAction Stop
            Write-Debug "User $user is valid"
            set-licenseoption
        } catch {
            Write-Warning "User $user is not valid... perhaps you entered a SMTP address and not a UPN. I will check --"
            try {
                $filter = "EmailAddress -like '" + $user + "'"
                $user = (Get-ADUser -Filter $filter -ErrorAction Stop).samaccountname + "@sullivanenterprises.org"
                Get-MsolUser -UserPrincipalName $user -ErrorAction Stop
                Write-Debug "User $user is valid"
                set-licenseoption
            } catch {
                Write-Debug "Failed check again"
                Write-Error "Can't find $user by email, and failed a upn check... exiting."
            }
        }
    }
{% endhighlight %}

This will loop through an array of values passed in the Identity parameter. First, we check to see if we were passed a UPN or a samaccountname. Ideally, I would add a check to see, first, if there is an @ symbol, and then validate that the correct domain name is included,  but for a start, this will work. In the case that we passed a samaccountname (i.e., no domain), we will append our domain to the username. 

Then, we proceed to attempt to query Microsoft Online for the user. Currently, any error will throw us to the catch block. Further development would suggest we should only go here for a user not found error, and catch other errors to a generic message. If we find the user, we know it is valid and we will run a function (in this case, 'set-licenseoption'). 

If an error is thrown, we will write a warning (logged with a module loaded earlier). A common reason why we might have an error is that someone grabbed the "UPN" out of Outlook because the UPN appears like an email address. To test that, we will build a filter (mainly because filtering as part of the Get-ADUser displays weird behavior with variables) and we will use that for the next line. IN the next line, we try to get the user account by filtering for any users with email addresses like the "UPN" entered. If we have a match or a __blank__ (which we know because we didn't have an error), we will proceed to query samaccountname@domain.com as the UPN once again. If we succeed this time, we proceed to running our function. If not, we will write an error.

Why are we okay with a blank with the get-aduser command? Because it will self-correct on the get-msoluser cmdlet and skip our function if we have no match. 

All in all, this was a quick solution to a problem. There are certainly improvements that can be made, but this will add a level of robustness to our scripts. 