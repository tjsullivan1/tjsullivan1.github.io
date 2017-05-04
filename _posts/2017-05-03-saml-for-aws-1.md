---
layout: post
title: "SAML for AWS Part 1"
subtitle: "AD FS Config"
author: "Tim Sullivan"
image: "post-bg.jpg"
---

# The Challenge

With my history in identity management, I like to avoid creating identities is multiple systems if there isn't a need for them. Luckily, working in larger enterprises, we often have tools to federate identities across multiple systems, allowing for easy management of accounts and access from one central space. In my case, we use Active Directory Federation Services (ADFS3.0), which utilizes SAML2.0 to perform identity assertions to various service providers. One of the more recent and exciting configurations was setting up the SAML connection between our multiple AWS accounts and our AD FS services. In the spirit of full disclosure, I may have missed some of my sources for this, and I relied heavily on Google for certain aspects of this script (e.g., '$AWS_Account_ID = Get-EC2SecurityGroup -GroupName "default" \| select -ExpandProperty OwnerID'). Some sources that I do have documented are:

* [https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_saml.html](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_saml.html)
* [https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_saml.html](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_saml.html)
* [http://www.padisetty.com/2014/02/powershell-automation-to-give-aws.html](http://www.padisetty.com/2014/02/powershell-automation-to-give-aws.html) (this one helped to build my do it all in one script)
* [https://cloudar.be/awsblog/enabling-federation-to-aws-using-windows-active-directory-adfs-and-saml/#configuringAWS](https://cloudar.be/awsblog/enabling-federation-to-aws-using-windows-active-directory-adfs-and-saml/#configuringAWS) (this one was crucial for the basics)

# The Solution 

The full solution can be found [here](https://gist.github.com/tjsullivan1/e6a9ea8871820d4d807d76eda6e281cd)

In order for this to work, we need to have the AWS PowerShell SDK installed (I did this on my AD FS server) and we need to initialize it. If we don't have either of those, we will error the script and exit. 

The first action we take is here:
{% highlight Powershell %} 
<#
###
Create AWS IAM Identity Provider
###
#>

# This puts all of our metadata into a varaiable for usewith the SAMLMetadataDocument parameter
$ADFS_Metadata_XML = Invoke-WebRequest $ADFS_Metadata_URL | select -ExpandProperty content

# We will need account_arn later for our policy document
New-IAMSAMLProvider -Name $IDP_Provider_Name -SAMLMetadataDocument $ADFS_Metadata_XML -OutVariable account_arn

{% endhighlight %}
These commands will download your AD FS metadata into a variable, and then attach that metadata to a newly created IAM identity provider within your AWS account (n.b., if you plan to do this for multiple accounts, you will need to re-initialize your AWS defaults to use the other account credentials and re-run the beginning of this script). We export the output of the New-IAMSAMLProvider cmdlet to the variable $account_arn. The output will look like this: arn:aws:iam::012345678901:saml-provider/ADFS.

Next, we create a policy doucment in JSON to allow users to take AWS roles with their federated accounts. The here string is used to make it keep the JSON formatting and to prevent problems with escaping quotes, etc. We replace the here string value with the previously captured Amazon resource name of the SAML provider. Then, we create a IAM role in AWS (this can be repeated by running this PowerShell [script](https://gist.github.com/tjsullivan1/2191e1b037172e3f02e466a533fe4f66)). It is critical that the role name match the end part of the Active Directory group we will create later.

Finally, we get into the AD FS side of things, which is a fairly standard set of claim rules. The authroization is set to allow all users (this can be changed later). The transorm is what actually sends the assertion to AWS. These were specified in the documentation listed above, and are mostly straightforward. The final two use custom rules, but we are getting all of the AD groups via the "Get AD Groups" rule and then sending that to a temporary variable which is referenced by the "Roles" rule. This allows us to have a group starting with "AWS-" (from the rule language: "Value =~ "(?i)^AWS-"]") and map that to an IAM role. For example, if I have a Role "Admin" in AWS, this can be accessed by anyone who is a member of the Active Directory group "AWS-Admin". The rule language works because we are replacing the AD value "AWS-" with the AWS friendly "arn:aws:iam::012345678901:saml-provider/ADFS,arn:aws:iam::012345678901:role/", indicating we are really looking for a role called: "arn:aws:iam::012345678901:role/Admin" with a trusted entity of: "arn:aws:iam::012345678901:saml-provider/ADFS". 

There is one trick to getting this to work across multiple accounts that I want to call out. The script as laid out will only work for one AWS account, but that can be remedied after creation. First, it is worth understanding that AD FS will not let you create two separate relying party trusts for your different accounts (because the AWS SAML endpoint has the same identifier). Two things will need to be changed; neither of which are in AWS. The first is our custom rule language for the "Roles" rule. We should modify that to look like this:
{% highlight Powershell %} 
c:[Type == "http://temp/variable", Value =~ "(?i)^AWS-([^d]{12})"]
 => issue(Type = "https://aws.amazon.com/SAML/Attributes/Role", Value = RegExReplace(c.Value, "AWS-([^d]{12})-", "arn:aws:iam::$1:saml-provider/ADFS,arn:aws:iam::$1:role/"));
{% endhighlight %}
This is telling us to find groups whose name matches a string starting with "AWS-" followed by 12 digits, followed by a final hyphen. We will then perform the same replacement as above, but it will sub in the individual account IDs and their matching roles. This allows us to have separate admins for accounts 012345678901 and 012345678902 by having a group "AWS-012345678901-Admin" and "AWS-012345678902-Admin".

Congratulations! Once you have run this script, you should be able to login to the AWS Console from your ADFS server's IdP initiated sign on page (should be here: https://\<ADFS_SERVER_FQDN\>/adfs/ls/IdpInitiatedSignOn.aspx).