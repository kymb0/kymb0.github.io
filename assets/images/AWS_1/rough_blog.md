After identifying a knowledge gap in pentesting AWS, I decided to spin up 


AWSgoat is an intentionally vulnerbale AWS lab environment with multiple paths to privilege escalation


sign up to app
login
browse to new post
explore file upload feature (confirm ssrf, attempt metadata v1 attack - doesnt work due to app being lambda)
use ssrf to download a copy of /proc/self/environ to retrieve secrets for lambda account
export env vars
discover the account does not have IAM list privileges
explore s3
list dev-bucket contents
grab ssh keys and config
ssh in and enumerate current roles policies
discover that the previous assumed role has full access on lambda and the current assumed role has createpolicy and attachrolepolicy
use these privileges to attach the 'dev-ec2-lambda-policies' policy to the 'blog_app_lambda_data' role which will allow for a privilege escalation vector via create lambda function
back in the session as blog_app_lambda_data, create a lambda function that will create a new policy and attach it to the blog_app_lambda_data role. We perform these actions inside a lmbda function to avoid running the priv esc commands via AWS cli