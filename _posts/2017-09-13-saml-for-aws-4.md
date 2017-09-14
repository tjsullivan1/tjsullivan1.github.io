---
layout: post
title: "SAML for AWS Part 4"
subtitle: "Revisiting the aws_saml_auth script"
author: "Tim Sullivan"
image: "post-bg.jpg"
---


# The Challenge

After several months of using the aws_saml_auth script (based on the [samlapi_formauth_adfs3 script from AWS](https://s3.amazonaws.com/awsiammedia/public/sample/SAMLAPICLIADFS/samlapi_formauth_adfs3.p)), there were several areas that were clearly lacking. Some of these included the fact that it didn't have parameters, meaning you had to modify the source code each time you wanted to tweak your credential location, profile name, etc. I still haven't parameterized everything, but this is much closer to a production ready script in my mind. Another feature request that came from some of my peers was adding a friendly name instead of the ARN to the indexed list of roles (example below).
{% highlight python %}
Please choose the role you would like to assume:
[ 0 ]:  arn:aws:iam::012345678901:role/S3FullAccess
[ 1 ]:  arn:aws:iam::123456789012:role/S3FullAccess
[ 2 ]:  arn:aws:iam::012345678901:role/ReadOnly
[ 3 ]:  arn:aws:iam::123456789012:role/Admin
[ 4 ]:  arn:aws:iam::012345678901:role/Admin
Selection:
{% endhighlight %}

# The Solution

I have a github repository with the full code that you can star/watch [here](https://github.com/tjsullivan1/aws_saml_auth).

First, I began to parameterize using a Python package [plac](https://pypi.python.org/pypi/plac). By wrapping the entire script in a "login" function, I was able to use parameters by calling that function with plac: `plac.call(login)`. The actual parameters themselves are included in the function defintion:
{% highlight python %}
def login(username: ("Your user-principal-name", 'option', 'u'),
          profile: ("The name of the profile we want to put it the credential file", 'option', 'p')='default',
          region: ("The AWS region", 'option', 'r')='us-east-2',
          filename: ("The filename that contains a comma separated mapping of account ids to friendly names",
                     'option', 'f')='account_ids.txt'):
{% endhighlight %}

plac lets us run aws_saml_auth.py with parameters such as "--username tjsullivan1@sullivanenterprises.org" or "-u tjsullivan1@sullivanenterprises.org", which gives us much more flexibility to quickly adjust things on the fly. Note that parameters without an = after the parentheses do not have a default option and will be prompted if not provided. All in all, plac seems to be a really useful and I will likely use it as a python script addition in the future.

As you can see, a couple of other useful parameters are profile. For instance, if you want to use a 'saml' profile, you can specify that here. You can have multiple profiles created from this script at once, which can allow for a lot of command line flexibility.

Second, I wanted to change `arn:aws:iam::012345678901:role/S3FullAccess` to something more like `SE-Test/S3FullAccess`, so that fellow admins who aren't in our accounts every day don't need to remember that 012345678901 is the test account. This was achieved with two major functions, seen here.

{% highlight python %}
def make_friendly_name(saml_role_response, account_name, account_id):
    """Replace the AWS friendly format with user friendly format
    Example:
    Convert arn:aws:iam::012345678901:role/Admin to Prod/Admin
    make_friendly_name('arn:aws:iam::012345678901:role/Admin', 'Prod', '012345678901')

    Keyword arguments:
    saml_role_response -- This is an AWS friendly role name, e.g. arn:aws:iam::012345678901:role/Admin
    account_name       -- This is the user friendly name for the account, e.g. "Prod"
    account_id         -- The AWS account id 012345678901
    """
    saml_role_response = re.sub(account_id, account_name, saml_role_response).replace(
        'arn:aws:iam::', '').replace(':role', '')
    return saml_role_response


def return_friendly_name_from_file(saml_role_response, filename):
    """Replace the AWS friendly format with user friendly format, using file map
    Example:
    Convert arn:aws:iam::012345678901:role/Admin to Prod/Admin
    return_friendly_name_from_file('arn:aws:iam::012345678901:role/Admin', 'account_ids.txt')

    Keyword arguments:
    saml_role_response -- This is an AWS friendly role name, e.g. arn:aws:iam::012345678901:role/Admin
    filename           -- This is the name of a file that contains comma separated values, e.g. account_ids.txt
    """
    tmp_account_id = re.sub(
        r'\/.*', '', saml_role_response).replace('arn:aws:iam::', '').replace(':role', '')
    if os.path.exists(filename):
        with open(filename, 'r') as account_file:
            found = False
            for line in account_file:
                if re.search(tmp_account_id, line):
                    good_line = line
                    found = True
            if found:
                account_id, account_name = good_line.rstrip().split(',')
                if account_id in saml_role_response:
                    return make_friendly_name(saml_role_response, account_name, account_id)
            else:
                return saml_role_response
    else:
        return saml_role_response
{% endhighlight %}

The first function replaces a specified account id with a specified account name in an arn string like the one listed above. It does this by using a regular expression substition and then removing the aws default ARN info. That is to say, the first piece (re.sub()) takes the string arn:aws:iam::012345678901:role/S3FullAccess and changes it to arn:aws:iam::SE-Test::role. Then, we replace the arn:aws:iam:: with a null string, and then we replace :role with a null value.

The second function reads a file line by line. If the file contains the account ID returned by the saml response, we flag that it was found and then we grab the name that we want assigned to the account from that line. We then return the response value of the make_friendly_name function -- if the line does not contain the value, we will just return what we used to return. Also, if the file provided doesn't exist, we return the value that we used to return.

With these two functions and the addition of plac, the script came a long way to being much more usable.
