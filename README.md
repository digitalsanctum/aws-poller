# aws-poller
The following describes the setup for an experiment to determine the IP addresses of a specific AWS service over time. 

In this case, I'd like to know the IP addresses associated with DynamoDB in the us-east-1 region. A shell script runs on an EC2 instance in an endless loop. For each iteration of the loop, we call `dig` and combine the result with various other metadata we're interested in. We then persist in a DynamoDB table to be queried later and perhaps do things like cross-reference the IP address and timestamp with Flow Log records.

Although this is a crude implementation, there are no dependencies outside of an EC2 instance and an attached IAM role which should have permissions to update the DynamoDB table.


## Create DynamoDB Table

Use the follow aws cli command to create the table

```
aws dynamodb create-table --table-name aws.dynamodb.ips \
 --key-schema '[{"AttributeName":"id","KeyType":"HASH"},{"AttributeName":"dt","KeyType":"RANGE"}]' \
 --attribute-definitions '[{"AttributeName":"dt","AttributeType":"S"},{"AttributeName":"id","AttributeType":"S"}]' \
 --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1
``` 

Or, use the following CloudFormation (JSON) resource snippet:

```
{
  "Type" : "AWS::DynamoDB::Table",
  "Properties" : {
    "TableName" : "aws.dynamodb.ips",
    "AttributeDefinitions" : [ {
      "AttributeName" : "dt",
      "AttributeType" : "S"
    }, {
      "AttributeName" : "id",
      "AttributeType" : "S"
    } ],
    "KeySchema" : [ {
      "AttributeName" : "id",
      "KeyType" : "HASH"
    }, {
      "AttributeName" : "dt",
      "KeyType" : "RANGE"
    } ],
    "ProvisionedThroughput" : {
      "ReadCapacityUnits" : 1,
      "WriteCapacityUnits" : 1
    }
  }
}
```
Or, use the following CloudFormation (YAML) resource snippet:

```
Type: "AWS::DynamoDB::Table"
Properties:
  TableName: "aws.dynamodb.ips"
  AttributeDefinitions:
  - AttributeName: "dt"
    AttributeType: "S"
  - AttributeName: "id"
    AttributeType: "S"
  KeySchema:
  - AttributeName: "id"
    KeyType: "HASH"
  - AttributeName: "dt"
    KeyType: "RANGE"
  ProvisionedThroughput:
    ReadCapacityUnits: 1
    WriteCapacityUnits: 1
```    

## dig.sh

Create `/home/ec2-user/dig.sh` with the following:
```
#!/usr/bin/env bash

set -e

TBL="aws.dynamodb.ips"
SVC="dynamodb"
REG="us-east-1"
ENDPOINT="${SVC}.${REG}.amazonaws.com"
MAC=$(curl -s http://169.254.169.254/latest/meta-data/network/interfaces/macs/)
VPC=$(curl -s http://169.254.169.254/latest/meta-data/network/interfaces/macs/${MAC}/vpc-id/)

while [ 1 ]
do
  ID=$(uuidgen)
  DT=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
  # TODO combine these
  IP=$(dig +nocmd +noall +answer ${ENDPOINT} @169.254.169.253 | awk '{print $5}' &)
  TTL=$(dig +nocmd +noall +answer ${ENDPOINT} @169.254.169.253 | awk '{print $2}' &)
  wait

  SECONDS=0
  AWS_CMD=$(aws dynamodb update-item --table-name 'aws.dynamodb.ips' \
  --key "{ \"id\": {\"S\":\"${ID}\"},\"dt\": {\"S\":\"${DT}\"}}" \
  --update-expression "SET #ip = :ip, #svc = :svc, #reg = :reg, #vpc = :vpc" \
  --expression-attribute-names "{\"#ip\":\"ip\", \"#svc\": \"svc\", \"#reg\": \"reg\", \"#vpc\": \"vpc\"}" \
  --expression-attribute-values "{\":ip\":{\"S\":\"${IP}\"}, \":svc\":{\"S\":\"${SVC}\"}, \":reg\":{\"S\":\"${REG}\"}, \":vpc\":{\"S\":\"${VPC}\"}}" \
  --return-values ALL_NEW --region us-west-2 >> dig.log)
  SLEEP_PERIOD=$(expr $TTL - $SECONDS)
  sleep $SLEEP_PERIOD
done
```
Make it executable: `chmod +x dig.sh`

## Install and Configure CloudWatch Logs Agent

For observability, you can optionally pipe the response of the `update-item` requests to CloudWatch logs.

For install instructions, follow: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/QuickStartEC2Instance.html

To configure, update `/etc/awslogs/awslogs/conf` with something like:

```
[/home/ec2-user/dig]
datetime_format = %b %d %H:%M:%S
file = /home/ec2-user/dig.log
buffer_duration = 5000
log_stream_name = {instance_id}
initial_position = start_of_file
log_group_name = /home/ec2-user/dig
region=us-west-2
```
If you want to send CW logs to a region other than us-east-1 (the default), then update `/etc/awslogs/awscli.conf`:

```
[plugins]
cwlogs = cwlogs
[default]
region = us-west-2
```

To restart awslogs on Amazon Linux 2: `sudo systemctl start awslogsd`

## TODO
- Add DynamoDB conditional update expression to prevent duplicate IPs within the same predetermined time period.
- Parameterize the aws service and region.
- Run in parallel from multiple regions and availability zones for more comphrehensive data.
