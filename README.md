# aws-poller

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
  echo $TTL
  echo $SECONDS
  SLEEP_PERIOD=$(expr $TTL - $SECONDS)
  echo $SLEEP_PERIOD
  echo $IP
  echo ""
  sleep $SLEEP_PERIOD
done
```
Make it executable: `chmod +x dig.sh`

## Install and Configure CloudWatch Logs Agent

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
