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

We use this connection string to access Azure storage by installing the Azure Storage extension in Visual Studio code, right clicking and selecting `Attach Storage Account`, then in the prompt for a connection string paste `"CON_STR"` value that was extracted from `/home/site/wwwroot/local.settings.json`.  

![sh_keys](/assets/images/azure/ssh_keys.jpg)  

We use the extracted `.ssh config` and keys to ssh to an Azure endpoint.

![ssh_session](/assets/images/azure/ssh_as_justin.jpg)  

Now that we have a foothold in the targets Azure environment, we start by seeing if we can view resources with `az resource list` which we can, we will then run `az role assignment list -g azuregoat_app` to list the role assignments that exist at a resource group scope. So, basically, we are enumerating who has access to the `azuregoat_app` resource group.  

_Azure role-based access control (Azure RBAC) is the authorization system you use to manage access to Azure resources. To determine what resources users, groups, service principals, or managed identities have access to, you list their role assignments._
https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-list-cli

### The first two screenshots are outputs of interest from listing resources when compared against listing role assignments, which is exhibited in the third screenshot. 

#### Screenshot 1
![resourcelist_vm](/assets/images/azure/get_vmname_princ_id.jpg)  

#### Screenshot 2
![resourcelist_automation_account](/assets/images/azure/owner_princ_id.jpg)   

#### Screenshot 3
![role_assignments](/assets/images/azure/role_assignments.jpg)   

It appears an automation account has owner privileges over the entire azuregoat_app resource group.  
A brief readthrough on [Microsoft's](https://learn.microsoft.com/en-us/azure/automation/overview#process-automation) knowlwedge page on Automation Accounts concedes the below information:  

![automation_purpose](/assets/images/azure/automation_account_purpose.jpg) 

This looks like an attractive vector for any malcious actor, and we will 

which opens up a vector for escalation using `runbooks`, which are blocks of either pre-defined or user-defined code used for the purpose of automation.



## Abusing Runbooks to Escalate Privileges  
Similar to how we abused Lambda in AWSGoat to escalate our privileges, we will write a block of Powershellworkflow and save it to the target machine as `escalation.ps1` code before creating, updating, publishing, and finally, running the runbook.

![runbook](/assets/images/azure/escalation_ps1.jpg)   

```
az automation runbook create --automation-account-name "dev-automation-account-appazgoat531441" -g azuregoat_app --name "dev-vm-test" --type "PowerShellWorkflow" --location "East US"
az automation runbook replace-content --automation-account-name dev-automation-account-appazgoatxxxxxx --resource-group azuregoat_app --name dev-vm-test --content @escalation.ps1
az automation runbook publish --automation-account-name dev-automation-account-appazgoatxxxxxx --resource-group azuregoat_app --name dev-vm-test
az automation runbook start --automation-account-name dev-automation-account-appazgoatxxxxxx1 --resource-group azuregoat_app --name dev-vm-test
```
![runbook](/assets/images/azure/runbook_replace_publish_start.jpg)  


We can see that we successfuly escalated our role to `owner` over the entire `azuregoat_app` resource group by again running `az role assignment list -g azuregoat_app` and seeing that the principal id of the virtual machine (which we are executing under the context of) now has the owner role assigned.

![runbook](/assets/images/azure/escalation_success.jpg)  

Now with these
