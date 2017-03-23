---
layout: post
title: "Get Identity from Multiple Values"
subtitle: "Building on Last Week"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# The Challenge

Last week, I discussed a validation for an identity parameter. I discussed potentially expanding the functionality of that to be a more reusable tool. I have found that people often find it is difficult to gather specific information for identity operations, and this script will make it easier to gather based on limited info as well as performing transformations from one property type (samaccountname) to another (userprincipalname or employeeid). 

# The Solution 

Below is an initial draft of a more robust script than last week's: [Get-Identity](https://gist.github.com/tjsullivan1/2d0123c5151897c69f4d3c4671d12883)

# The Breakdown

As usual, I start this script with a cmdletbinding invocation to indicate that I want to be able to use the PowerShell standard parameters. Then, we go into our parameters specific to this script. First, we have the $Identity string, which will allow multiple values so that we can pass a list or a whole file to this script for parsing. Second, we have the output type. By default, we will return samaccountname (username), but the script will allow us to grab any AD property. 

Next, we move into the BEGIN block. In this block, we create all of our functions to be used later. The reason for the BEGIN block is that it is run only once when multiple values are in the identity parameter (NOT multiple values passed to the identity parameter).

The first function builds a filter string to be reused in the get-aduser query (I have had poor luck with variables in a filter string, and this is an easy way to avoid it). This function simply concatenates two strings into one string.
{% highlight Powershell %}
    function Create-Filter {
        param(
            $filter_type,

            $filter_spec
        )

        $filter = $filter_type + $filter_spec
        return $filter
    }
{% endhighlight %}

The second function will find an AD user based on the filter that we create with the create-filter function. Note that by default, the filter will be built ending in "-eq '$user'" where $user is the value that we are passing to this script. In this function, we build a filter string and try to find a user based on that value. This is a standard AD cmdlet function, just built for simplicity and replicability. There is a good amount of debugging and verbosity built into this script so we can test and watch what happens (as well as potentially logging).
{% highlight Powershell %}
    function Get-User {
         param(
            $filter_type,

            $filter_spec = " -eq '" + $user + "'"
        )

        try {
            $filter = create-filter -filter_type $filter_type -filter_spec $filter_spec
            if (get-aduser -Filter $filter -ErrorAction Stop) {
                Write-Debug "Successfully found user $user based on $filter_type" 
                Write-Verbose "Successfully found user $user based on $filter_type"

               $found = get-aduser -Filter $filter -Properties * -ErrorAction Stop
               return $found
            } else {
                Write-Debug "No results found for $user searching with $filter_type"
            }
        } catch {
            Write-Debug "Error in search for $user using $filter_type"
        }
    }
{% endhighlight %}

Finally in the BEGIN block, we have the meat of this script. This creates an array based on a sequential most specific to least specifc search criteria. When you unpeel the onion, we are running a test with the previous function continuously to create a new filter based on that test. For instance, we will search for a user with based on what was passed to the identiy parameter (e.g. tim.sullivan@sullivanenterprises.org) in the samaccountname. Obviously, an experienced admin will recognize that isn't a valid samaccountname, but remember this script can perform a transformation. In that case, we will build a filter that looks like this "samaccountname -eq 'tim.sullivan@sullivanenterprises.org'". This search will return null, which is equivalent to false in PowerShell. Therefore, we will move on to the next statement (elseif), where we wil return a result. In that case, the results array is appended and returned. 

The other values are self explanatory, but worth noting are displayname and searching by name. In either of thes cases, it is worth noting that there could be multiple users found. 

{% highlight Powershell %}
    function Search-UsersByFilter {
        $results = @()

        if (get-user -filter_type samaccountname) {
            $results += get-user -filter_type samaccountname
        } elseif (get-user -filter_type emailaddress) {
            $results += get-user -filter_type emailaddress
        } elseif (get-user -filter_type userprincipalname) {
            $results += get-user -filter_type userprincipalname
        } elseif (get-user -filter_type employeeid) {
            $results += get-user -filter_type employeeid
        } elseif (get-user -filter_type DisplayName) {
            $results += get-user -filter_type DisplayName
        } elseif ($user -like "* *") {
            Write-Debug "Trying a search based on first and last name"
            $fname = $user.split(' ')[0]
            $lname = $user.split(' ')[-1]
            $p1 = 'GivenName -like "' + $fname + '*"'
            $p2 = '-and Surname -like "' + $lname + '*"'

            # Note the debug/verbose output will be a little weird for this because we aren't using an actual filter type.
            $results += get-user -filter_type $p1 -filter_spec $p2
        }

        return $results
    }
{% endhighlight %}

Now, we move on to the PROCESS block. This block is run against every value in the identity parameter. This block utilizes each of the functions we declared above (because they all call eachother). The only meat of this is that it calls the search-usersbyfilter function, and if more than one result is returned we warned the admin. If no results are returned, we warn the admin. If one result is returned we are happy :). 

{% highlight Powershell %}
PROCESS {
    foreach ($user in $Identity) {
        Write-Verbose $user
 
        $results = Search-UsersByFilter
                
        if ($results.Count -gt 1) {
            Write-Warning "Multiple results were found for your search for $user"
            $results | % {
                Write-Debug "Multiple results"
                $result = $_
                $result.$output_type
            }
        } elseif ($results) { # if there is only one result, it isn't an array so .count will not return a value
            Write-Debug "One result for $user"
            $results.$output_type
        } else {
            Write-Warning "No results were found for $user"
            Write-Debug "pause"
        }
    }
}
{% endhighlight %}

After all of this, you may be wondering... how do I do a transformation? Well, by default, output_type will return a samaccountname, but if we had a list of samaccountnames and we wanted their employeeids, we would run get-identity -Identity (my_list_of_samaccountnames.txt) -output_type EmployeeID.