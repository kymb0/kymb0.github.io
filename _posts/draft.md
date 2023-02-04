---
layout: single
title: "Azuredly attacking Azure..."
excerpt: "Using AzureGoat to better understand how to attack Azurte"
date: 2023-02-04
header:
  thumb: /assets/images/azure/LeAzure.jpg
  teaser: /assets/images/azure/LeAzure.jpg
  teaser_home_page: true
classes: wide
categories:
  - Cloud
tags:
  - Azure
---


![LeMe_LeAzure](/assets/images/azure/LeAzure.jpg)

For the second part in our attacking cloud series, we will attack [AWSGoat](https://github.com/ine-labs/AzureGoat) which is an intentionally vulnerable Azure lab environment with multiple paths to privilege escalation.   

Just real quick before we get into this: I was looking for some puns to start this blog off with however upon looking up the definition of azure I discovered the below.  

![where_are_they](/assets/images/azure/where_are_the_clouds.jpg)  

Azure means clear sky and NO clouds??? Anyway, let's move on.  


## Discovering the Application

Once we successfully deploy our environment via terraform, we will have access to the application URL, and navigating here shows a blog website. This is much the same as what we did in [Part 1](https://kymb0.github.io/IAM-attacking-AWS-rn/) where we attacked AWS.  

![blog_landing_page](/assets/images/azure/blog.jpg)

Again, we abuse sign up feature that allows us to create our own account to gain access to a dashboard where we can create new blog posts.  

![register](/assets/images/azure/signup.jpg)  
![new_post](/assets/images/azure/newpost.jpg)  

After poking around realising that the app appears to be more or less identical to what we saw on AWSGoat, we once again seek to exploit the file upload feature, as the fact we can upload a file via presenting a URL indicates that either the server will be embedding a link to the image on the blog, or retrieving the data at the specified URL and storing it somewhere. This will present a clear vector once again for SSRF.  

![file_upload_feature](/assets/images/AWS_1/fileupload.jpg)  

## Getting a Foothold

Where the attack chain differs from what we did previously against AWSGoat (we retrieved `/proc/self/environ`), is the local file we seek to retrieve. This time we will try to retrieve `/home/site/wwwroot/local.settings.json`  

As per [Microsoft](https://learn.microsoft.com/en-us/azure/azure-functions/functions-develop-local#local-settings-file)  
![localsettings](/assets/images/azure/local_settings.jpg)  

![ssrf_localsettings_lfi](/assets/images/azure/ssrf_local_settings.jpg)  

We download the stored file and `cat` the contents, revealing a connection string and secrets:  

![ssrf_localsettings_loot](/assets/images/azure/ssrf_local_settings_loot.jpg)    

We use this connection string to access Azure storage by installing the Azure Storage extension in Visual Studio code, right clicking and selecting `Attach Storage Account`, then in the prompt for a connection string paste `"CON_STR"` value that was extracted from /home/site/wwwroot/local.settings.json.  

![sh_keys](/assets/images/azure/ssh_keys.jpg)  

We use the extracted `.ssh config` and keys to ssh to an Azure endpoint.

![ssh_session](/assets/images/azure/ssh_as_justin.jpg)  

Unfortunately from here however, we discover the account does not have IAM list privileges after trying to list policies. I even tried to enumerate with an AWS exploit tool called [PACU](https://github.com/RhinoSecurityLabs/pacu) to see if I was missing anything, it was however a dead-end on the IAM front. When this happened I said "IAM disappointed"  

![dissapointed](/assets/images/AWS_1/dissapointed.gif)  
![list_policy_fail](/assets/images/AWS_1/list_policy_fail.jpg)  
![pacu_IAM_enum_fail](/assets/images/AWS_1/pacu_iam_enum_fail.jpg)  

## Exploring S3

We know that our file is being stored on the production S3 bucket, so we run `aws s3api list-buckets` to see what other buckets there are and how much access we have. After revealing a potentially interesting bucket, we attempt to list the objects and find some rather interesting files. It appears that this bucket has been used to store sensitive information pertaining to users accessing the AWS environment via SSH.

![s3_list-buckets](/assets/images/AWS_1/s3_enum_dev_bucket.jpg)  
![s3_list_objects](/assets/images/AWS_1/s3_list_objects_loot.jpg)  

Although we were unable to enumerate our own level of permissions, we can determine we have a high level of access to the `S3` service as not only can we list these objects, but we can access and download them as well.

![s3_download_objects](/assets/images/AWS_1/s3_download_loot.jpg)
![s3_stolen_config](/assets/images/AWS_1/ssh_config.jpg)

The stolen information in the exposed bucket allow us to SSH straight into an EC2 instance which did not have proper [rules for inbound traffic](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/authorizing-access-to-an-instance.html) setup.  

![ssh_successful](/assets/images/AWS_1/ssh_successful.jpg)

## Reviewing IAM

A quick explanation of why we run `aws sts get-caller-identity` followed by `aws iam list-attached-role-policies --role-name AWS_GOAT_ROLE` after connecting via `ssh` is as follows: First, `sts get-caller-identity`, allows us to discover which credentials are being used to call `aws` operations, we can determine the specifics of what we are dealing with by matching the output against the `IAM` `ARN` syntaxes from the [AWS reference page](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_identifiers.html)

SO after looking at this, we know that the _assumed role_ syntax is as below, and that the rolename must be `AWS_GOAT_ROLE`, as the _role-session-name_ is simply used [uniquely identify](https://docs.aws.amazon.com/cli/latest/reference/sts/assume-role.html#:~:text=Use%20the%20role%20session%20name,account%20that%20owns%20the%20role.) a session when the role is assumed.

![ssh_succesful](/assets/images/AWS_1/understanding_output_of_getcalleridentity.jpg)  

Running `aws iam list-attached-role-policies --role-name AWS_GOAT_ROLE` will return the names and ARNs of the managed policies attached to the IAM role, which in this case as we have determined, is `AWS_GOAT_ROLE`. Think of this as kind of like enumerating which Active Directory groups a user is a member of.    

We retrieve the policy document for the `dev-ec2-lambda-policies` policy which will show us the actions the policy allows us to perform and which resources we can perform them against. Note we specified the version in our command, this can be retrieved with `aws iam get-policy --policy-arn`

![dev-ec2-lambda-policies](/assets/images/AWS_1/dev-ec2-lambda-policies.jpg)

So reviewing the above output we can see that this policy has access to attach policies to other roles, this is granted by the `iam:AttachRolePolcy` action.  
The resource it can perform this action against is restricted to `blog_app_lambda_data` which just so happens to be the other role we have access to via the previously explored SSRF attack. This policy also has the `iam:CreatePolicy` action set, which presents a very interesting escalation vector - allow me to explain below:  
If we are looking at a policy right now to determine what this role can and cannot do, and through doing so we have discovered that the policy itself allows for the creation of NEW polices, and that we can attach those policies to the `blog_app_lambda_data` role which we have access to, this means that we can grant a policy allowing for ALL actions against ALL resources, so basically Administrator access without actually attaching the AWS managed `AdministratorAccess` policy, which, in a real setting would likely raise alarm bells if we were to add ourselves to it.

## Abusing Lambda to escalate privileges

### Think of Lambda as "Function as a Service", it is based on a "micro-VM" architecture called [Firecracker](https://github.com/firecracker-microvm/firecracker)

I decided to go down the route of abusing these actions via Lambda, as I thought [Cloudtrail](https://docs.aws.amazon.com/cli/latest/reference/cloudtrail/index.html) may be less likely to be configured to feed Lambda logs into [GuardDuty](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_data-sources.html) in a "real" environment (I could be way off here), nevertheless the following is a valuable exercise in understanding IAM and Lambda.  
Let's take another look at the `blog_app_lambda_data` role now that we have an account with sufficient `iam` actions to effectively enumerate.

![dev-ec2-lambda-policies](/assets/images/AWS_1/lambda-data-role-policies.jpg)

The `blog_app_lambda_data` role has full lambda access, meaning we can create functions. If we look back at `dev-ec2-lambda-policies` we see that `iam:PassRole` is present. The `iam:PassRole` allows for the creation of functions which will execute under the context of another role.  
With the above in mind, if the `blog_app_lambda_data` role had the same level of permissions that the `AWS_GOAT_ROLE` role did through the `dev-ec2-lambda-policies` policy, we could effectively create a lambda function that creates a new policy and attaches it to the `blog_app_lambda_data` role.  
We could then abuse the presence of `iam:PassRole` to execute a new lambda function with the `dev-ec2-lambda-policies` level of permissions.  
Luckily the `iam:AttachRolePolcy` action will allow us to simply attach the `dev-ec2-lambda-policies` policy to `blog_app_lambda_data`, we do this as below:  

![blog_app_lambda_data_attachrole](/assets/images/AWS_1/blog_app_lambda_data_attachrole.jpg)  

Now that we have the permissions of both policies attached to one role, let's create a malicious lambda function as below:  

![lambda_priv_esc_function](/assets/images/AWS_1/lambda_priv_esc_function.jpg)  

We zip the `.py` file up and create a lambda function before invoking it. Running `aws iam list-attached-role-policies --role-name blog_app_lambda_data` after doing this shows that we have successfully managed to create a new policy and attach it, all from within a lambda function.  

**IMPORTANT NOTE** _Although for the purpose of this exercise I opted to perform these actions from a local terminal with the exported secrets, best practice would be to run commands from an EC2 instance wherever possible so that it doesn't look like actions are being performed from outside AWS. so, in other words, I should have created, invoked, and deleted the function from my ssh session via exporting the secrets as was done earlier in this post_  

![lambda_priv_esc_success](/assets/images/AWS_1/lambda_priv_esc_success.jpg)  

Don't forget to remove the function afterwards:  

![cleanup](/assets/images/AWS_1/cleanup.jpg)  

We can double check the actions we can perform and the resources we can perform them against via our new policy, and as we can see, it is quite permissive (an asterisk denotes any action/resource)  

![full_perms](/assets/images/AWS_1/full_perms.jpg)  

So after this reading this post, you should no longer have your head in the clouds with regards to `IAM` at the very least.  

### STAY TUNED FOR THE NEXT EPISODE  
![tv](/assets/images/AWS_1/adventure_time_tv.jpg)  
