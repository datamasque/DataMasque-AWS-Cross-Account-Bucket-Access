# Cross-Account Access to S3 Buckets for masking runs.

## Introduction

This repository contains CloudFormation templates for deploying IAM roles and policies necessary to configure access between the DataMasque application (running on EC2 instances or EKS clusters) and AWS S3 buckets. Comprehensive scenarios covering various patterns are documented here. - https://datamasque.com/portal/documentation/latest/file-connections.html#configuring-access-between-datamasque-and-aws-s3-buckets

### Prerequisites

- AWS CLI configured with the appropriate credentials for the target AWS account.
- A DataMasque instance Role ARN and Name, for ec2 deployments or EKS Role Name/Arn for EKS deployments

The bucket names and prefixes used in these example scenarios are for illustration purposes only. Please update them to match your environment's configurations.

## Scenario-1: Granting DataMasque Application Access to Source and Destination Buckets within the Same AWS Account.
The CloudFormation template deploys an IAM policy and attaches it to the IAM role assigned to the EC2 instance running the DataMasque application. This setup enables the DataMasque application to mask data on source S3 buckets and write the masked data to destination S3 buckets. If the source and target S3 buckets are within the same AWS account as the DataMasque application, no additional configuration steps are needed.



Required Cloudformation parameters: 
  - RoleName: The IAM role attached to the EC2 instance running the DataMasque application.
  - SourceBuckets: Comma-separated ARNs of the source S3 buckets where data needs to be masked.
  - DestinationBuckets: Comma-separated ARNs of the target S3 buckets where masked data will be written.
  - SourceBucketsPrefixes: Comma-separated ARNs of the source S3 buckets with prefixes indicating where the data to be masked is stored.
  - DestinationBucketsPrefixes: Comma-separated ARNs of the destination S3 buckets with prefixes indicating where the masked data needs to be stored.

```shell
export DmRoleName=DataMasque-Role
export DestinationBucketsArns=arn:aws:s3:::dest-bucket1,arn:aws:s3:::dest-bucket2
export SourceBucketsArns=arn:aws:s3:::source-bucket1,arn:aws:s3:::source-bucket2
export DestinationBucketsPrefixes=arn:aws:s3:::dest-bucket1/masked_data,arn:aws:s3:::dest-bucket2/masked_data
export SourceBucketsPrefixes=arn:aws:s3:::source-bucket1/unmasked/credit_card_data,arn:aws:s3:::source-bucket2/unmasked/user_data
aws cloudformation create-stack \
  --stack-name datamasque-aws-account-s3bucket-access \
  --template-body file://datamasque-s3-bucket-access-iam-policy.yaml \
  --parameters \
        ParameterKey=PolicyName,ParameterValue=datamasque-aws-account-policy \
        ParameterKey=DestinationBuckets,ParameterValue=\"${DestinationBucketsArns}\" \
        ParameterKey=SourceBuckets,ParameterValue=\"${SourceBucketsArns}\" \
        ParameterKey=SourceBucketsPrefixes,ParameterValue=\"${SourceBucketsPrefixes}\" \
        ParameterKey=DestinationBucketsPrefixes,ParameterValue=\"${DestinationBucketsPrefixes}\" \
        ParameterKey=DmRoleName,ParameterValue=${DmRoleName} \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND
```

## Scenario 2: Granting DataMasque Application Access to Source and Destination Buckets in Different AWS Accounts Using Cross-Account IAM Roles


This scenario requires deploying the CloudFormation stack (`datamasque-crossaccount-access`) in the AWS account where the DataMasque application is running, as well as the CloudFormation stack (`datamasque-crossaccount-s3bucket-access`) in the AWS account where the buckets are configured.

The `datamasque-crossaccount-access` stack deploys an IAM policy containing sts:AssumeRole permissions and attaches it to the IAM role assigned to the EC2 instance running the DataMasque application.

The `datamasque-crossaccount-s3bucket-access` stack deploys an IAM role that can be assumed by the IAM role attached to the EC2 instance running the DataMasque application. It also deploys policies that allow the DataMasque application to perform masking operations on the specified S3 buckets.

Required parameters for Cloudformation stack `datamasque-crossaccount-access`: 
  - DmRoleName: IAM role name attached to ec2 instance running DataMasque application.
  - CrossAccountRoles: Comma-separated ARNs of IAM roles deployed in the AWS accounts where the source buckets, which require data masking, are configured.

```shell
export DmRoleName=DataMasque-Role
export CrossAccountRoles=arn:aws:iam::2222222222:role/datamasque-s3bucket-access-role,arn:aws:iam::3333333333:role/datamasque-s3bucket-access-role,arn:aws:iam::4444444444:role/datamasque-s3bucket-access-role
aws cloudformation create-stack \
  --stack-name datamasque-crossaccount-access \
  --template-body file://datamasque-crossaccount-access.yaml \
  --parameters \
        ParameterKey=PolicyName,ParameterValue=datamasque-crossaccount-policy \
        ParameterKey=CrossAccountRoles,ParameterValue=\"${CrossAccountRoles}\" \
        ParameterKey=DmRoleName,ParameterValue=${DmRoleName} \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND

```

Required parameters for Cloudformation stack `datamasque-crossaccount-s3bucket-access`: 
  - DmRoleName: IAM role `ARN` attached to ec2 instance running DataMasque application.
  - CrossAccountRole: IAM role name assumed by the DataMasque application to perform data masking operations on S3 buckets.
  - SourceBuckets: Comma-separated ARNs of the source S3 buckets where data needs to be masked.
  - DestinationBuckets: Comma-separated ARNs of the target S3 buckets where masked data will be written.
  - SourceBucketsPrefixes: Comma-separated ARNs of the source S3 buckets with prefixes indicating where the data to be masked is stored.
  - DestinationBucketsPrefixes: Comma-separated ARNs of the destination S3 buckets with prefixes indicating where the masked data needs to be stored.

This CloudFormation stack should be deployed in the AWS account where the S3 buckets are configured.

```shell
export DmRoleName=DataMasque-Role
export DestinationBucketsArns=arn:aws:s3:::dest-bucket1,arn:aws:s3:::dest-bucket2
export SourceBucketsArns=arn:aws:s3:::source-bucket1,arn:aws:s3:::source-bucket2
export DestinationBucketsPrefixes=arn:aws:s3:::dest-bucket1/masked_data,arn:aws:s3:::dest-bucket2/masked_data
export SourceBucketsPrefixes=arn:aws:s3:::source-bucket1/unmasked/credit_card_data,arn:aws:s3:::source-bucket2/unmasked/user_data
export CrossAccountRole=arn:aws:iam::2222222222:role/datamasque-s3bucket-access-role
aws cloudformation create-stack \
  --stack-name datamasque-crossaccount-s3bucket-access \
  --template-body file://datamasque-crossaccount-s3bucket-access.yaml \
  --parameters \
        ParameterKey=CrossAccountRole,ParameterValue=\"${CrossAccountRole}\" \
        ParameterKey=DestinationBuckets,ParameterValue=\"${DestinationBucketsArns}\" \
        ParameterKey=SourceBuckets,ParameterValue=\"${SourceBucketsArns}\" \
        ParameterKey=SourceBucketsPrefixes,ParameterValue=\"${SourceBucketsPrefixes}\" \
        ParameterKey=DestinationBucketsPrefixes,ParameterValue=\"${DestinationBucketsPrefixes}\" \
        ParameterKey=DmRoleName,ParameterValue=${DmRoleName} \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND

```


## Scenario-3: Granting DataMasque Application Access to Source and Destination Buckets in Different AWS Accounts Using Bucket Policies

In this scenario, when the DataMasque application and the source/destination buckets are configured in different AWS accounts, in addition to deploying the CloudFormation stack from Scenario-1, you will also need to apply the following bucket policies to the respective source and destination buckets.

Source Bucket policy
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "SourceBucketRead",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::111111111111:role/DM-Role"
                ]
            },
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::<bucket-name>",
                "arn:aws:s3:::<bucket-name>/*"
            ]
        },
        {
            "Sid": "SourceBucketPermissionCheck",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::111111111111:role/DM-Role"
                ]
            },
            "Action": [
                "s3:ListBucket*",
                "s3:GetBucketAcl",
                "s3:GetBucketPolicyStatus",
                "s3:GetBucketPublicAccessBlock"
            ],
            "Resource": [
                "arn:aws:s3:::<bucket-name>",
            ]
        },       
        {
            "Sid": "AllowSSLRequestsOnly",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::<bucket-name>",
                "arn:aws:s3:::<bucket-name>/*"
            ],
            "Condition": {
                "Bool": {
                    "aws:SecureTransport": "false"
                }
            }
        }
    ]
}

```


Destination Bucket policy

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DestinationBucketWrite",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::111111111111:role/DM-Role"
                ]
            },
            "Action": [
                "s3:PutObject",
                "s3:GetObject*"
            ],
            "Resource": [
                "arn:aws:s3:::<bucket-name>",
                "arn:aws:s3:::<bucket-name>/*"
            ]
        },
        {
            "Sid": "DestinationBucketSecurityCheck",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::111111111111:role/DM-Role"
                ]
            },
            "Action": [
                "s3:ListBucket*",
                "s3:GetBucketAcl",
                "s3:GetBucketPolicyStatus",
                "s3:GetBucketObjectLockConfiguration"
            ],
            "Resource": [
                "arn:aws:s3:::<bucket-name>"
            ]
        },              
        {
            "Sid": "AllowSSLRequestsOnly",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::<bucket-name>",
                "arn:aws:s3:::<bucket-name>/*"
            ],
            "Condition": {
                "Bool": {
                    "aws:SecureTransport": "false"
                }
            }
        }
    ]
}
```
Please replace `arn:aws:iam::111111111111:role/DM-Role` with the actual role name attached to EC2 instance running DataMasque application.
