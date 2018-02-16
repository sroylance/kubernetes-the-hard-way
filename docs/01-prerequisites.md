# Prerequisites

## Amazon web services
Install the AWS CLI
log in to your AWS account and generate an access and secret key for your account by clicking on 'My Security Credentials' on the account drop down
run aws configure and, following the prompts, enter your preffered region and the keys retrieved from the UI
the following creates an IAM user we will use for the remainder of the labs.  It has unlimited access to a number of AWS services, so use caution.

```
aws iam create-group --group-name hardway
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name hardway
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name hardway
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name hardway
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name hardway
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name hardway
aws iam create-user --user-name hardway
aws iam add-user-to-group --user-name hardway --group-name hardway
aws iam create-access-key --user-name hardway
```

run aws configure again, with the keys returned above

generate an SSH key pair with ssh-keygen and upload it
```
aws ec2 import-key-pair --key-name hardway --public-key-material "`ssh-keygen -y -f ~/.ssh/id_rsa`"
```

Next: [Installing the Client Tools](02-client-tools.md)
