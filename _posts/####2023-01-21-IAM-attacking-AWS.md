---
layout: single
title: "IAM hacking AWS"
excerpt: "Using AWSGoat to better understand how to attack AWS"
date: 2023-01-21
header:
  thumb: /assets/images/AWS_1/chillin_on_cloud.jpg
  teaser: /assets/images/AWS_1/chillin_on_cloud.jpg
  teaser_home_page: true
classes: wide
categories:
  - Cloud
tags:
  - AWS
  - IAM
---


![chillin_on_a_cloud](/assets/images/AWS_1/chillin_on_cloud.jpg)

After identifying a knowledge gap in pentesting AWS, I decided to spin up and attack [AWSGoat](https://github.com/ine-labs/AWSGoat) which is an intentionally vulnerable AWS lab environment with multiple paths to privilege escalation.  

I learnt a ton about IAM and how to attack it and, more importantly, had an absolute blast.  

Enough chit-chat, leeeeet's jump riiiggghhht into AWS!  

## Discovering the Application

Once we succesfuly deploy our environment via terraform, we will have access to the application URL, and navigating here shows a blog website.  
![blog_landing_page](/assets/images/AWS_1/Blog_landing_page.jpg)

We do not have any credentials at this stage, there is however, a sign up feature that allows us to create our own account to gain access to a dashboard where we can create new blog posts.    
![register](/assets/images/AWS_1/Sign_up.jpg)  
![new_post](/assets/images/AWS_1/newpost.jpg)  

The first thing that jumps out at us is the file upload feature, and to be more precise, the fact that we can upload via a url. This indicates that either the server will be embeding a link to the image on the blog, or retrieving the data at the specified URL and storing it somewhere. Hopefully it is the later, as this will present a clear vector for SSRF.  
![file_upload_feature](/assets/images/AWS_1/fileupload.jpg)  

So to test this, we feed it a page containing an image, and view the response in burp to better understand the application logic:

![retrieved_data_stored_on_s3](/assets/images/AWS_1/upload_url_feature.jpg)  

We visit the returned s3 link to confirm it contains the original linked image and lo and behold, the best case scenario is true, and we are able to make requests in some capacity within the context of the server (aka Server Side Request Forgery, an often overlooked vulnerability)  
![CATE](/assets/images/AWS_1/kitten_image.jpg)  

## Getting a Foothold

The most obvious step from here is to attempt a metadata v1 attack, where we trick the underlying EC2 instance to hit the AWS metadata endpoint to retrive privileged information.

![ssrf_v1_attempt](/assets/images/AWS_1/ssrf_v1_attempt.jpg)  

This however does not work and returns a server error immediately. Further testing revealed that the application required a `200` response to be succesful. This indicates either that metadata v2 was in use which requires more sophisticated techniques, or, that the webapp may be running on lambda.  

After trying a few different approaches, I discovered an LFI (local file inclusion) escalation which allowed for the retrieval of files within the ephemeral environment.  
As a common way to feed credentials to lambda functions is via environment variables, we attempt to continue our chain of attacks by retrieving a copy of `/proc/self/environ`  

![ssrf_4_real](/assets/images/AWS_1/SSRF.JPG)  

We download the stored file on the S3 bucket and `cat` the contents, revealing the AWS secrets of the role running the application:  

![ssrf_loot](/assets/images/AWS_1/SSRF_LOOT.JPG)  

We store these secrets in our envrinment variables for our terminal session and succesfully make a call to AWS:  

![stolen_key_auth_success](/assets/images/AWS_1/stolen_key_auth_success.jpg)  

Unfortunately from here however, we discover the account does not have IAM list privileges after trying to list policies. I even tried to enumerate with an AWS exploit tool called [PACU](https://github.com/RhinoSecurityLabs/pacu) to see if I was missing anything, it was however a dead-end on the IAM front. When this happened I said "IAM dissapointed"  

![dissapointed](/assets/images/AWS_1/dissapointed.gif)  
![list_policy_fail](/assets/images/AWS_1/list_policy_fail.jpg)  
![pacu_IAM_enum_fail](/assets/images/AWS_1/pacu_iam_enum_fail.jpg)  

## Exploring S3

We know that our file is being stored on the production S3 bucket, so we run `aws s3api list-buckets` to see what other buckets there are and how much access we have. After revealing a potentially interesting bucket, we attempt to list the objects and find some rather interesting files. It appears that this bucket has been used to store sensitive information pertaining to users accessing the AWS environment via SSH.

![s3_list-buckets](/assets/images/AWS_1/s3_enum_dev_bucket.jpg)  
![s3_list_objects](/assets/images/AWS_1/s3_list_objects_loot.jpg)  

Although we were unable to enumerate our own level of permissions, we can determine we have a high level fo access to the `S3` service as not only can we list thes objects, but we can access and download them as well.

![s3_download_objects](/assets/images/AWS_1/s3_download_loot.jpg)
![s3_stolen_config](/assets/images/AWS_1/ssh_config.jpg)

The stolen information in the exposed bucket allow us to SSH straight into an EC2 instance which did not have proper [rules for inbound traffic](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/authorizing-access-to-an-instance.html) setup.  

![ssh_succesful](/assets/images/AWS_1/ssh_successful.jpg)

## Reviewing IAM

A qucik explaination of why we run `aws sts get-caller-identity` followed by `aws iam list-attached-role-policies --role-name AWS_GOAT_ROLE` after sonnecting via `ssh` is as follows: First, `sts get-caller-identity`, allows us to discover which credentials are being used to call `aws` operations, we can determine the specifics of what we are dealing with by matching the output against the `IAM` `ARN` syntaxes from the [AWS reference page](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_identifiers.html)

SO after looking at this, we know that the _assumed role_ syntax is as below, and that the rolename must be `AWS_GOAT_ROLE`, as the _role-session-name_ is simply used [uniquely identify](https://docs.aws.amazon.com/cli/latest/reference/sts/assume-role.html#:~:text=Use%20the%20role%20session%20name,account%20that%20owns%20the%20role.) a session when the role is assumed.

![ssh_succesful](/assets/images/AWS_1/understanding_output_of_getcalleridentity.jpg)  

Running `aws iam list-attached-role-policies --role-name AWS_GOAT_ROLE` will return the names and ARNs of the managed policies attached to the IAM role, which in this case as we have determined, is `AWS_GOAT_ROLE`. Think of this as kind of like enumerating which Active Directory groups a user is a member of.    

We retrieve the policy document for the `dev-ec2-lambda-policies` policy which will show us the actions the policy allows us to perform and which resources we can perform them against. Note we specified the version in our command, this can be retrieved with `aws iam get-policy --policy-arn`

![dev-ec2-lambda-policies](/assets/images/AWS_1/dev-ec2-lambda-policies.jpg)

So reviewing the above output we can see that this policy has access to attach policies to other roles, this is granted by the `iam:AttachRolePolcy` action. The rourcse it can perform this action against is restriced to `blog_app_lambda_data` which just so happens to be the other role we have access to via the previously explored SSRF attack. This policy also has the `iam:CreatePolicy` action set, which presents avery interesting escalation vector - allowe me to explain: if we are looking at a policy right now to determine what this role can and cannot do, and through doing so we have discovered that the policy itself allows for the creation of NEW polcies, and that we can attach those policies to the `blog_app_lambda_data` role which we have access to, this means that we can grant a policy allowing for ALL actions against ALL resources, so basically Administrator access without actually attaching the AWS managed `AdministratorAccess` polciy, which, in a real setting would likely raise alarm bells if we were to add ourselves to it.

## Abusing Lambda to escalate privileges

It would be good to avoid running the commands straight from the CLI if possible, again, to avoid leaving behind any obvious atrifacts of privilege escalation. Let's take another look at the `blog_app_lambda_data` role now that we have an account with sufficient `iam` actions to effectively enumerate.

![dev-ec2-lambda-policies](/assets/images/AWS_1/lambda-data-role-policies.jpg)

The `blog_app_lambda_data` role has full lambda access, meaning we can create functions. If we look back at `dev-ec2-lambda-policies` for we see that `iam:PassRole` is present. If the `blog_app_lambda_data` role had the same level of permissions that the `AWS_GOAT_ROLE` role did through the `dev-ec2-lambda-policies` policy; we could effectively create a lambda function that creates a new policy and attaches it to the `blog_app_lambda_data` role. We could abuse the  presence of `iam:PassRole` to execute with the `dev-ec2-lambda-policies` level of permissions.  
Luckily the `iam:AttachRolePolcy` action will allow us to simply attach the `dev-ec2-lambda-policies` policy to `blog_app_lambda_data`, we do this as below:  

![blog_app_lambda_data_attachrole](/assets/images/AWS_1/blog_app_lambda_data_attachrole.jpg)

Now that we have the permissions of both policies attached to one role, let's create a malicous lambda function as below:  

![lambda_priv_esc_function](/assets/images/AWS_1/lambda_priv_esc_function.jpg)  

We zip the `.py` file up and create a lambda function before invoking it. Running `aws iam list-attached-role-policies --role-name blog_app_lambda_data` after doing this shows that we have succesfully managed to create a new policy and attach it, all from within a lambda function.  

![lambda_priv_esc_success](/assets/images/AWS_1/lambda_priv_esc_success.jpg)  

Don't forget to remove the function afterwards:  
![cleanup](/assets/images/AWS_1/cleanup.jpg)
