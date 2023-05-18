# Analyze Elastic IP usage

Elastic IP address (EIP) is a static public IPv4 address allocated to your Amazon EC2 instance or other AWS resources such as LoadBalancers that you can reach via the internet. EIP addresses are designed for dynamic cloud computing because they can re-attach to another instance if their existing instance fails for resiliency. These EIPs are used for applications that must make external requests to services that require allowlisted-inbound connections. You are charged per EIP not associated with a running instance per hour. As an application scales up and down, these EIPs may occasionally be used a few times over the course of weeks or months. Over time, unused EIPs can accumulate and add unnecessary costs to your AWS bill. 

In this blog post, I will show you how to analyze EIP usage history to have a better insight of your EIP usage pattern in your You can use this solution regularly as part of your cost optimization effort to safely remove unused EIPs to reduce your costs. 

## Solution overview

This solution uses AWS CloudTrail and Amazon Athena to analyze historical EIP attachment activity in your AWS account. AWS CloudTrail and Amazon Athena help make it easier by combining the detailed CloudTrail log files with the power of the Athena SQL engine to easily find, analyze, and respond to changes and activities in an AWS account. 

The solution compares the snapshot of the current EIPs and looks for their most recent attachment within the last three month (date can be customized). it then determines how frequently EIPs were attached to resources. If the attachments are greater than zero that means these EIPs are actively being used. If the attachment count is zero that means these EIPs are running idle and can be released to save costs..

## Prerequisite 

1. Create an Athena table - The first step is to create an Athena table for a CloudTrail trail using the CloudTrail console. AWS CloudTrail is a service that records AWS API calls and events for Amazon Web Services accounts. 

    1. Open the [CloudTrail console](https://console.aws.amazon.com/cloudtrail/home). In the navigation pane, choose Event history. b. Choose Create Athena table.
    1. For Storage location, use the down arrow to select the Amazon S3 bucket where log files are stored for the trail to query.
    1. Choose Create table. The table is created with a default name that includes the name of the Amazon S3 bucket.
    1. Copy table name in notepad as it is required for the next step. It should name like cloudtrail_logs_***

2. Install the AWS CLI - This post uses AWS Command Line Interface (CLI) examples. To utilize these, you must first install and configure the AWS CLI. For more information, see Installing [the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

## Implementation

AWS CloudFormation template URL and parameters - In this section, we use AWS CloudFormation to create required resources. AWS CloudFormation is a service that helps you model and set up your AWS resources so that you can spend less time managing those resources and more time focusing on your applications that run in AWS. 

The CloudFormation template creates Athena views to search past AssociateAddress events in CloudTrail, AWS Lambda function to collect existing EIPs, and an S3 bucket for storing the analysis result. 

Run the following commands in a terminal. These commands clone the GitHub repository and deploy the CloudFormation stack to your AWS account

**NOTE** - In CloudFormation parameters, replace TableName with the table name created from step 1 in prerequisites.

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
    --parameters ParameterKey=CloudTrailAthenaTableName,ParameterValue='TABLENAME'
```
Wait for the stack to be created. It should take a few minutes to complete. You can open CloudFormation console to view stack creation process.

## Run analysis

We have configured all prerequisites in order to run our analysis. Now, we need to run the below steps to analyze EIP attachment history.

1. In your command line terminal, run the below CLI commands.

```
# A) Get Lambda function name from CFN output
FUNCTION=`aws cloudformation describe-stacks --stack-name EIP-analyzer | jq -r '.Stacks[0].Outputs[] | select( .OutputValue | contains("EIPlistFinder"))' | jq -r ".OutputValue"`

# B) Invoke the Lambda to list the current EIPs
aws lambda invoke --function-name $FUNCTION out --log-type Tail --query 'LogResult' --output text |  base64 -d
```
2. Login in the Athena Console and copy and paste the following DDL statement into the Athena console query editor and choose Run query.

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

This post demonstrated how you can analyze Elastic IP usage history to have a better insight of EIP attachment pattern. You can regularly use this analysis as part of your cost optimization strategy to identify and release inactive EIPs to reduce costs.
