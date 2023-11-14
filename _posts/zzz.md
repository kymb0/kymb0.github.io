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
  - Cloud Engineering
tags:
  - AWS
  - Cross-Account
  - RDS
  - Lambda
---

![Cross-Account_Access](/assets/images/AWS_cross_account/cross_account_header.jpg)

I know this is not the normal subject matter for my blog - however I was recently faced with an interesting challenge at work, and I identified a few knowledge gaps in AWS regarding RDS and cross account access and thus after a few beers with my [hackin homie](https://onecloudemoji.github.io/) an idea for a blog no one asked for and no one needs was once again incepted.

## BUT WHY

Basically, one account has a database in Amazon [RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html), and the other account needs to access it via a lambda for marketing information.  
It sounds like this should be straight forward, however cross-account RDS access in AWS can be tricky due to AWS's built-in security between accounts. The need for cross account access often comes up when you have different AWS accounts for dev and prod or when you need to share database resources between different teams or third-party vendors.
To enable interactions between RDS instances across AWS accounts, it's necessary to configure specific trust relationships, permissions, and networking routes.
This setup ensures that cross-account interactions are both controlled and in line with the [principle of least privilege (PoLP)](https://www.digitalguardian.com/blog/what-principle-least-privilege-polp-best-practice-information-security-and-compliance).



## Account Setup

We start by creating two user accounts, `aws_dev_account` (under account *2974) and `aws_engineering_account` (under account *5884). The first account, `aws_dev_account`, has the following policies attached:

- **AmazonRDSFullAccess**: Grants full access to Amazon RDS, necessary for managing RDS instances where our data resides.
- **AmazonVPCFullAccess**: Essential for setting up VPC peering to allow connectivity between the two AWS accounts.
- **IAMFullAccess**: Allows for the creation and management of IAM roles and policies, crucial for setting up cross-account access.

![dev_perms](/assets/images/AWS_cross_account/dev_perms.png)

The second account, `aws_engineering_account`, has the following policies attached:

- **CrossAccountRDSAccessRole_ougoing**: This custom policy, ([created later in this guide](#cross-account-role-creation)), grants scoped permissions to assume the cross-account role, enabling access to a scoped in `aws_dev_account`.
- **AWSLambdaBasicExecutionRole**: Grants basic execution capabilities for Lambda functions, including logging via Amazon CloudWatch Logs, facilitating the data retrieval process.

![engineering_perms](/assets/images/AWS_cross_account/engineering_perms.png)



## Cross Account Role Creation:

Creating a cross-account role is a crucial step to allow different account development and engineering accounts. This role will specifically enable the engineering team to access the RDS instance in the development account while adhering to AWS's principle of least privilege (PoLP).

Login to `aws_dev_account`, `navigate to IAM Dashboard` > `Roles` > `Create role`.
Select `Another AWS account` under the `Select trusted entity` section and enter the account ID of `aws_engineering_account`.

![create_crossaccountrdsaccessrole](/assets/images/AWS_cross_account/create_crossaccountrdsaccessrole.png)

Edit the trust policy to restrict access to the specified account only.

![select_trusted_entity](/assets/images/AWS_cross_account/select_trusted_entity.png)


Create a custom policy in aws_engineering_account to allow assuming the CrossAccountRDSAccessRole, and attach it to the engineering user.
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::xxxxxxx5884:user/aws_engineering_account"
            },
            "Action": "sts:AssumeRole",
            "Condition": {}
        }
    ]
}
```

![edit_trust_policy](/assets/images/AWS_cross_account/edit_trust_policy.png)

Now, log in as the admin in `aws_engineering_account`, create a custom policy to allow this account to assume the `CrossAccountRDSAccessRole` role, and attach it to the engineering user.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::xxxxxx2974:role/CrossAccountRDSAccessRole"
        }
    ]
}
```
![specify_assumerole_perms](/assets/images/AWS_cross_account/specify_assumerole_perms.png)


## VPC Peering Setup

Initiate VPC peering by navigating to: `VPC Dashboard` > `Peering Connections` > `Create peering connection` in either account. Fill out the details and select the VPC that your RDS is hosted on. Once done, log into the other account, go to `VPC Dashboard` > `Peering Connections` to accept the connection request. Update the route tables in both accounts to allow traffic flow: `VPC Dashboard` > `Route Tables` > select the route table associated with your requester VPC, and add a route to the peering connection.

![create_peering](/assets/images/AWS_cross_account/create_peering.png)

![vpc_peering_requested](/assets/images/AWS_cross_account/vpc_peering_requested.png)

![accept_peering_request](/assets/images/AWS_cross_account/accept_peering_request.png)




## Security Group Configuration

Ensure a security group allows inbound traffic from the originating VPC CIDR block. Navigate to: `VPC Dashboard` > `Security Groups` > `Create Security Group`, and specify the inbound rules to allow traffic from the other VPC.

## Secrets Manager Setup

In `aws_dev_account`, create a secret in AWS Secrets Manager to store the database credentials. During secret creation, at the *Resource permissions* step, specify that the `CrossAccountRDSAccessRole` can access the secret.

![setup_secrets_manager](/assets/images/AWS_cross_account/setup_secrets_manager.png)
![blog_secret_resource_permissions](/assets/images/AWS_cross_account/blog_secret_resource_permissions.png)

```json
{
  "Version" : "2012-10-17",
  "Statement" : [ {
    "Effect" : "Allow",
    "Principal" : {
      "AWS" : "arn:aws:iam::aws_dev_account:role/CrossAccountRDSAccessRole"
    },
    "Action" : "secretsmanager:GetSecretValue",
    "Resource" : "arn:aws:secretsmanager:us-east-1:aws_dev_account:secret:db-credentials"
  } ]
}
```


![specify_perms_secretmanager](/assets/images/AWS_cross_account/specify_perms_secretmanager.png)


## Role Assumption and Data Retrieval

As `aws_engineering_account`, assume the cross-account role using the following one-liner:

```bash
eval $(aws sts assume-role --role-arn arn:aws:iam::aws_dev_account:role/CrossAccountRDSAccessRole --role-session-name MySession --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' --output text | awk '{print "aws configure set aws_access_key_id "$1"; aws configure set aws_secret_access_key "$2"; aws configure set aws_session_token "$3";"}')
```
![assumerole_evidence](/assets/images/AWS_cross_account/assumerole_evidence.png)

Create and run Python scripts to fetch data from the RDS instance. Below is a script to retrieve vegetables data:

```import mysql.connector
import boto3
import json
import logging
from decimal import Decimal

# Set up logging to stdout for script, disable logging for boto
logging.basicConfig(level=logging.ERROR, handlers=[logging.StreamHandler()])
logging.getLogger('boto3').setLevel(logging.ERROR)
logging.getLogger('botocore').setLevel(logging.ERROR)

# Custom serialization function to handle Decimal values
def default(obj):
    if isinstance(obj, Decimal):
        return float(obj)
    raise TypeError("Object of type '%s' is not JSON serializable" % type(obj).__name__)

def main():
    try:
        # Get credentials from Secrets Manager
        secrets_client = boto3.client('secretsmanager', region_name='us-east-1')
        secret_value_response = secrets_client.get_secret_value(
            SecretId='arn:aws:secretsmanager:us-east-1:xxxxxx2974:secret:blog/read-veggies'
        )
        db_credentials = json.loads(secret_value_response['SecretString'])

        # Database credentials
        db_username = db_credentials['username']
        db_password = db_credentials['password']

        # Connect to the database
        conn = mysql.connector.connect(
            host='fruit-n-veg-db.xxxxxxx.us-east-1.rds.amazonaws.com',
            user=db_username,
            password=db_password,
            database='fruits_veggies'
        )

        # Execute SQL query
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM vegetables;")
        result = cursor.fetchall()

        # Print the response in JSON format
        result_json = [dict(zip(cursor.column_names, row)) for row in result]
        print(json.dumps(result_json, indent=2, default=default))

    except Exception as e:
        print(f'Error: {e}')

    finally:
        # Close the database connection
        conn.close()

if __name__ == "__main__":
    main()
```

![lambda success evidence](/assets/images/AWS_cross_account/lambda_success_evidence.png)
