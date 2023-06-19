# Analyze Elastic IP usage

Elastic IP address (EIP) is a static public IPv4 address allocated to your Amazon EC2 instance or other AWS resources such as LoadBalancers that you can reach via the internet. EIP addresses are designed for dynamic cloud computing because they can re-attach to another instance if their existing instance fails for resiliency. These EIPs are used for applications that must make external requests to services that require allowlisted-inbound connections. You are charged per EIP not associated with a running instance per hour. As an application scales up and down, these EIPs may occasionally be used a few times over the course of weeks or months. Over time, unused EIPs can accumulate and add unnecessary costs to your AWS bill. 

In this repo, I will show you how to analyze EIP usage history to have a better insight of your EIP usage pattern in your AWS account. You can use this solution regularly as part of your cost optimization effort to safely remove unused EIPs to reduce your costs. 

## Solution overview

This solution uses AWS CloudTrail and Amazon Athena to analyze historical EIP attachment activity in your AWS account. AWS CloudTrail and Amazon Athena help make it easier by combining the detailed CloudTrail log files with the power of the Athena SQL engine to easily find, analyze, and respond to changes and activities in an AWS account. 

The solution compares the snapshot of the current EIPs and looks for their most recent attachment within the last three month (date can be customized). it then determines how frequently EIPs were attached to resources. If the attachments are greater than zero that means these EIPs are actively being used. If the attachment count is zero that means these EIPs are running idle and can be released to save costs.

## Prerequisite 

1. Create an Athena table - The first step is to create an Athena table for a CloudTrail trail using [partition projection](https://docs.aws.amazon.com/athena/latest/ug/cloudtrail-logs.html). AWS CloudTrail is a service that records AWS API calls and events for Amazon Web Services accounts.

   The following example statement automatically uses partition projection on CloudTrail logs from a specified date until the present. 

```
CREATE EXTERNAL TABLE `CloudTrailAthenaTableEIPAnalyzer`(
  `eventversion` string COMMENT 'from deserializer', 
  `useridentity` struct<type:string,principalid:string,arn:string,accountid:string,invokedby:string,accesskeyid:string,username:string,sessioncontext:struct<attributes:struct<mfaauthenticated:string,creationdate:string>,sessionissuer:struct<type:string,principalid:string,arn:string,accountid:string,username:string>,ec2roledelivery:string,webidfederationdata:map<string,string>>> COMMENT 'from deserializer', 
  `eventtime` string COMMENT 'from deserializer', 
  `eventsource` string COMMENT 'from deserializer', 
  `eventname` string COMMENT 'from deserializer', 
  `awsregion` string COMMENT 'from deserializer', 
  `sourceipaddress` string COMMENT 'from deserializer', 
  `useragent` string COMMENT 'from deserializer', 
  `errorcode` string COMMENT 'from deserializer', 
  `errormessage` string COMMENT 'from deserializer', 
  `requestparameters` string COMMENT 'from deserializer', 
  `responseelements` string COMMENT 'from deserializer', 
  `additionaleventdata` string COMMENT 'from deserializer', 
  `requestid` string COMMENT 'from deserializer', 
  `eventid` string COMMENT 'from deserializer', 
  `readonly` string COMMENT 'from deserializer', 
  `resources` array<struct<arn:string,accountid:string,type:string>> COMMENT 'from deserializer', 
  `eventtype` string COMMENT 'from deserializer', 
  `apiversion` string COMMENT 'from deserializer', 
  `recipientaccountid` string COMMENT 'from deserializer', 
  `serviceeventdetails` string COMMENT 'from deserializer', 
  `sharedeventid` string COMMENT 'from deserializer', 
  `vpcendpointid` string COMMENT 'from deserializer', 
  `tlsdetails` struct<tlsversion:string,ciphersuite:string,clientprovidedhostheader:string> COMMENT 'from deserializer')
PARTITIONED BY ( 
  `account` string, 
  `region` string, 
  `day` string)
ROW FORMAT SERDE 
  'org.apache.hive.hcatalog.data.JsonSerDe' 
STORED AS INPUTFORMAT 
  'com.amazon.emr.cloudtrail.CloudTrailInputFormat' 
OUTPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  's3://BUCKET/'
TBLPROPERTIES (
  'projection.account.type'='enum', 
  'projection.account.values'='ACCOUNT_ID', 
  'projection.day.format'='yyyy/MM/dd', 
  'projection.day.interval'='1', 
  'projection.day.interval.unit'='DAYS', 
  'projection.day.range'='2023/01/01,NOW', 
  'projection.day.type'='date', 
  'projection.enabled'='true', 
  'projection.region.type'='enum', 
  'projection.region.values'='us-east-2,us-east-1,us-west-1,us-west-2,af-south-1,ap-east-1,ap-south-1,ap-northeast-3,ap-northeast-2,ap-southeast-1,ap-southeast-2,ap-northeast-1,ca-central-1,eu-central-1,eu-west-1,eu-west-2,eu-south-1,eu-west-3,eu-north-1,me-south-1,sa-east-1', 
  'storage.location.template'='s3://BUCKET/AWSLogs/${account}/CloudTrail/${region}/${day}', 
  'transient_lastDdlTime'='1686327817')
```
2. Replace the following clauses with your account information.

* In the LOCATION and storage.location.template clauses, replace BUCKET to point to the Amazon S3 bucket that contains your log data.
* In the projection.account.values, replace with your account-id.
* For projection.day.range, replace 2023/01/01 with the starting date that you want to use.

3. Login in the Athena Console and copy and paste the above DDL statement into the Athena console query editor and choose Run query.

4. Install the AWS CLI - This post uses AWS Command Line Interface (CLI) examples. To utilize these, you must first install and configure the AWS CLI. For more information, see Installing [the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

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
