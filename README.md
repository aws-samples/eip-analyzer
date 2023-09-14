# Analyze Elastic IP usage

AWS Elastic IP address (EIP) is a static public unique IPv4 address. Allocated exclusively to your AWS account, the EIP remains under your control until you decide to release it. It can be allocated to your Amazon EC2 instance or other AWS resources such as LoadBalancers. EIP addresses are designed for dynamic cloud computing because they can be re-mapped to another instance to mask any disruptions. These EIPs are also used for applications that must make external requests to services that require a consistent address for allowlisted-inbound connections. As your applications usage varies, these EIPs might see sporadic use over weeks or even months, leading to potential accumulation of unused EIPs that may inadvertently inflate your AWS expenditure.

In this blog post, I will show you how to analyze EIP usage history to have a better insight of your EIP usage pattern in your AWS account. You can leverage this solution regularly as part of your cost optimization effort to safely remove unused EIPs to reduce your costs. 
 
## Solution overview

This solution leverages activity logs from AWS CloudTrail and power of Amazon Athena to conduct comprehensive analysis of historical EIP attachment activity within your AWS account. AWS CloudTrail, a critical AWS Service, meticulously logs API activity within an AWS Account.

On the other hand, Amazon Athena is an interactive query service that simplifies data analysis in Amazon S3 using standard SQL. It is a serverless service, eliminating the need for infrastructure management and costing you only for the queries you execute.

By extracting detailed information from AWS CloudTrail and querying it using Athena, this solution streamlines the process of data collection, analysis, and reporting of EIP usage within an AWS account.

To gather EIP usage reporting, this solution compares snapshots of the current EIPs, focusing on their most recent attachment within a customizable three-month period. It then determines the frequency of EIP attachments to resources. An attachment counts greater than zero suggests that the EIPs are actively in use. In contrast, an attachment count of zero indicates that these EIPs are idle and can be released, aiding in identifying potential areas for cost reduction.

![eip1](https://github.com/aws-samples/eip-analyzer/assets/32849802/3d52556c-1a15-4185-a2aa-137ac061ae63)

## Prerequisite 

Install the AWS CLI - This post uses AWS Command Line Interface (CLI) examples. To utilize these, you must first install and configure the AWS CLI. For more information, see Installing [the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

## Implementation

In this section, we use AWS CloudFormation to create required resources. AWS CloudFormation is a service that helps you model and set up your AWS resources so that you can spend less time managing those resources and more time focusing on your applications that run in AWS. 

The CloudFormation template creates Athena views and table to search past AssociateAddress events in CloudTrail, AWS Lambda function to collect snapshot of existing EIPs, and an S3 bucket for storing the analysis result. 

Run the following commands in a terminal. These commands clone the GitHub repository and deploy the CloudFormation stack to your AWS account.

```
# A) Clone the repository and install jq
sudo yum install -y jq
git clone https://github.com/aws-samples/eip-analyzer.git

# B) Switch to the repository's directory
cd EIP-analyzer

# C) Create a stack to deploy all prerequisites 
aws cloudformation create-stack \
    --stack-name EIP-analyzer \
    --template-body file://template.yaml \
    --capabilities CAPABILITY_IAM \
    --parameters ParameterKey=CloudTrailS3Path,ParameterValue='REPLACE_CLOUDTRAIL_S3_LOCATION'
```
Wait for the stack to be created. It should take a few minutes to complete. You can open CloudFormation console to view stack creation process.

## Run analysis

We have configured all prerequisites in order to run our analysis. Now, we need to run the below steps to analyze EIP attachment history.

1. Run the CLI commands listed below in your command line terminal. These commands will invoke the Lambda function to retrieve a list of current EIPs in your account and save the results in an S3 bucket for analysis.

```
# A) Get Lambda function name from CFN output
FUNCTION=`aws cloudformation describe-stacks --stack-name EIP-analyzer | jq -r '.Stacks[0].Outputs[] | select( .OutputValue | contains("EIPlistFinder"))' | jq -r ".OutputValue"`

# B) Invoke the Lambda to list the current EIPs. 
aws lambda invoke --function-name $FUNCTION out --log-type Tail --query 'LogResult' --output text |  base64 -d
```
2. Login in the Athena Console and copy and paste the following DML statement into the Athena console query editor and choose Run query.

```
select 
eip.publicip,
eip.allocationid,
eip.region,
eip.accountid,
eip.associationid, 
eip.PublicIpv4Pool,
max(associate_ip_event.eventtime) as latest_attachment,
count(associate_ip_event.associationid) as attachmentCount
from eip LEFT JOIN associate_ip_event on associate_ip_event.allocationid = eip.allocationid 
group by 1,2,3,4,5,6
```

You can now run a query on CloudTrail logs to look back in time for Elastic IP attachment. This query provides you with better insight to safely release idle EIPs in order to reduce costs by displaying how frequently each specific EIP was previously attached to any resources.

This report will provide the following information:

- Public IP
- Allocation ID (the ID that Amazon Web Services assigns to represent the allocation of the Elastic IP address for use with instances in a VPC)
- Region
- Account-id
- latest_attachment date (last time EIP was attached to resource)
- attachment_count (Number of attachment)
- Association ID (The association ID for the address. If it is empty, EIP is idle not attached to any resources)

## Conclusion

This post demonstrated how you can analyze Elastic IP usage history to have a better insight of EIP attachment pattern. Please checkout GITHUB repo link to regularly run this analysis as part of your cost optimization strategy to identify and release inactive EIPs to reduce costs.
