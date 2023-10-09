---
layout: single
title: "Cross Account RDS Access"
excerpt: "Bridging the AWS Dev and Eng Accounts for Data Access"
date: 2023-10-12
header:
  thumb: /assets/images/aws/cross-account-thumb.jpg
  teaser: /assets/images/aws/cross-account-teaser.jpg
  teaser_home_page: true
  classes: wide
categories:
  - Engineering
tags:
  - AWS
  - Cross-Account
  - RDS
  - Lambda
---

![Header Image](/assets/images/aws/cross-account-header.jpg)

I know this is not the normal subject matter for my blog - however I was recently faced with an interesting challenge at work, and I identified a few knowledge gaps in AWS regarding RDS and cross account access and thus after a few beers with my [hackin homie](https://onecloudemoji.github.io/) an idea for a blog no one asked for and no one needs was once again incepted.

## BUT WHY

Cross-account RDS access in AWS can be tricky due to AWS's built-in security between accounts. The need for cross account access often comes up when you have different AWS accounts for dev and prod or when you need to share database resources between different teams or third-party vendors. Setting this up requires a good grasp of AWS IAM policies, role assumptions, and network configurations to make sure it's done securely and effectively.

## Setting up the Accounts

Initially, we set up two users in separate AWS accounts: `aws_dev_account` and `aws_engineering_account`. The former is the grill master with full control, while the latter is the hungry neighbor awaiting those delicious data ribs.

### Policies for aws_dev_account:

- **AmazonRDSFullAccess**: The all-access backstage pass to manage RDS instances.
- **AmazonVPCFullAccess**: The VIP ticket to the VPC concert, essential for setting up VPC peering.
- **IAMFullAccess**: The master key to the IAM kingdom.

### Policies for aws_engineering_account:

- **AmazonRDSReadOnlyAccess**: The reading glasses to view RDS resources and DB instances, but no touching!
- **AWSLambdaBasicExecutionRole**: The basic gear needed to run Lambda functions and pull data from RDS.

## Cross-Account Role Creation

In the `aws_dev_account`, we create a new role in IAM, selecting "another AWS account" as the Trusted Entity Type. We edit the Trust Policy to specify the `aws_engineering_account` as the trusted entity. This is like giving our neighbor a personalized invitation to our BBQ.

## VPC Peering

We initiate VPC peering from either account, ensuring the peering connection associates with the VPC hosting our RDS. Once the connection request is approved, we update the route tables in both accounts to allow traffic flow between them.

## Testing the Setup

In CloudShell as `aws_engineering_account`, we verify the successful role assumption with a convenient one-liner and a couple of screenshots showing `aws sts get-caller-identity` before and after the role assumption.

## AWS Secrets Manager Setup

We ensure our cross-account role can access the secret during the secret creation at the "Resource permissions" step. This is like handing over the BBQ sauce recipe but under a strict NDA.

## Running the Lambdas

We create two python scripts reflecting potential Lambdas, one for retrieving vegetables and one for fruits, and run them. And voila, the engineering team now has a taste of the data ribs!

![Vegetables Output](/assets/images/aws/vegetables-output.jpg)
![Fruits Output](/assets/images/aws/fruits-output.jpg)

## Conclusion

With this setup, we've securely bridged our Dev and Eng accounts, ensuring controlled access to essential data. Now, both teams can enjoy the BBQ while respecting each other's backyard!

**Note:** For a more fortified setup, in larger environments with an Aurora cluster, permissions could be further locked down in IAM with RDS:execute. This is like having a bouncer at the BBQ ensuring only the VIPs get the special sauce.

---

Enjoyed the read? Stay tuned for more cloud capers, and remember, a well-fed engineering team is a happy engineering team!

