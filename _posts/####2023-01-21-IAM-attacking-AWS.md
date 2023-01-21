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

Unfortunately from here however, we discover the account does not have IAM list privileges after trying to list policies. I even tried to enumerate with an AWS exploit tool called ![PACU](https://github.com/RhinoSecurityLabs/pacu) to see if I was missing anything, it was however a dead on the IAM front. When this happened I said "IAM dissapointed"  

![dissapointed](/assets/images/AWS_1/dissapointed.gif)  
![list_policy_fail](/assets/images/AWS_1/list_policy_fail.jpg)  
![pacu_IAM_enum_fail](/assets/images/AWS_1/pacu_iam_enum_fail.jpg)  

We know that our file is being stored on the production S3 bucket, so we run `aws s3api list-buckets` to see what other buckets there are and how much access we have. After revealing a potentially interesting bucket, we attempt to list the objects and find some rather interesting files. It appears that this bucket has been used to store sensitive information pertaining to users accessing the AWS environment via SSH.

![s3_list-buckets](/assets/images/AWS_1/s3_enum_dev_bucket.jpg)  
![s3_list_objects](/assets/images/AWS_1/s3_list_objects_loot.jpg)  

Although we were unable to enumerate our own level of permissions, we can determine we have a high level fo access to the `S3` service as not only ca we list thes objects, but we can access and download them as well

Now this is spciey, we cannot enumerate our own permissions in `IAM`, but we can not only list the `S3` buckets but can see what is inside of them, 


grab ssh keys and config
ssh in and enumerate current roles policies
discover that the previous assumed role has full access on lambda and the current assumed role has createpolicy and attachrolepolicy
use these privileges to attach the 'dev-ec2-lambda-policies' policy to the 'blog_app_lambda_data' role which will allow for a privilege escalation vector via create lambda function
back in the session as blog_app_lambda_data, create a lambda function that will create a new policy and attach it to the blog_app_lambda_data role. We perform these actions inside a lmbda function to avoid running the priv esc commands via AWS cli
