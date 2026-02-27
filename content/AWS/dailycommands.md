---
title: Daily Commands
tags:
  - AWS
enableToc: true
---
# EC2

## AMI

```shell
aws ec2 describe-images \
--profile <profile_id> \
--filters "Name=architecture,Values=x86_64" "Name=name,Values=*ubuntu*20.04*"


aws ec2 describe-images \
--profile <profile_id> \
--owners "099720109477" \
--filters \
"Name=architecture,Values=x86_64" \
"Name=name,Values=*ubuntu*24.04*" > ami.json
```

# S3

```shell
aws s3api list-buckets --profile prod --regio us-east-1

aws s3 cp main.py s3://busines-dev-uswe2-emr/scripts/iccid/main.py --profile <profile_id>

aws s3api create-bucket --bucket "polattoVMTestCLI" --region us-west-1 --create-bucket-configuration LocationConstraint=us-west-1
```

# SSM

```shell
aws ssm put-parameter --name "/ElasticAPM/server_url" --value "" --type "String"

aws ssm put-parameter --name "/ElasticAPM/secret_token" --value "" --type "SecureString"


aws ssm get-parameters --names "/ElasticAPM/server_url" "/ElasticAPM/secret_token"
```