# Data pipeline in AWS

## Requirements

* [AWS CLI installed](https://docs.aws.amazon.com/cli/latest/userguide/installing.html)
* [AWS SAM CLI installed](https://github.com/awslabs/aws-sam-cli)
* [Java SE Development Kit 8 installed](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
* [Docker installed](https://www.docker.com/community-edition)
* [Maven](https://maven.apache.org/install.html)

### Installing dependencies

Use `maven` to install dependencies and package applications into a JAR file:
```bash
$ mvn package
```

## Testing

Run the `JUnit` tests
```bash
$ mvn test
```

## Validate SAM template
```
$ sam validate --template template.yaml
```

## Deployment
```
$ Push built jar to S3 and run sam package/sam deploy as below.

```

## Package/deploy AWS resources
AWS CLI commands to package, deploy and describe outputs defined within the cloudformation stack:
```bash
$ sam package \
    --template-file template.yaml \
    --output-template-file packaged.yaml \
    --s3-bucket REPLACE_THIS_WITH_YOUR_S3_BUCKET_NAME

$ sam deploy \
    --template-file packaged.yaml \
    --stack-name data-pipeline-<env> \
    --capabilities CAPABILITY_IAM \
    --parameter-overrides MyParameterSample=MySampleValue

$ aws cloudformation describe-stacks \
    --stack-name data-pipeline-<env> --query 'Stacks[].Outputs'
```

## Enable PITR for dynamodb continuous backups (one off)

```
$ aws dynamodb update-continuous-backups \
--table-name TABLENAME  \
--point-in-time-recovery-specification PointInTimeRecoveryEnabled=True
```

## Configure Elasticsearch logging to a cloudwatch log group (one off)
* See [docs](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-createupdatedomains.html#es-createdomain-configure-slow-logs)
```bash
$ ES_LOG_GROUP_ARN=$(aws cloudformation describe-stacks --stack-name STACKNAME --query 'Stacks[0].Outputs[?OutputKey==`ESLogGroupArn`].OutputValue' --output text)
$ aws es update-elasticsearch-domain-config --domain-name ESDOMAIN \ 
                                            --log-publishing-options \
      "SEARCH_SLOW_LOGS={CloudWatchLogsLogGroupArn=$ES_LOG_GROUP_ARN,Enabled=true}, \
      INDEX_SLOW_LOGS={CloudWatchLogsLogGroupArn=$ES_LOG_GROUP_ARN,Enabled=true}, \ 
      ES_APPLICATION_LOGS={CloudWatchLogsLogGroupArn=$ES_LOG_GROUP_ARN,Enabled=true}"
$ aws logs put-resource-policy --policy-name my-policy --policy-document '{ "Version": "2012-10-17", "Statement": [{ "Sid": "", "Effect": "Allow", "Principal": { "Service": "es.amazonaws.com"}, "Action":[ "logs:PutLogEvents"," logs:PutLogEventsBatch","logs:CreateLogStream"],"Resource": "'$ES_LOG_GROUP_ARN'"}]}'
```

## Register curator snapshot repository for backup Lambda to use for ES (one off)
* See [docs](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-managedomains-snapshots.html#es-managedomains-snapshot-prerequisites)
```
$ ES_SNAPSHOT_BUCKET=$(aws cloudformation describe-stacks --stack-name STACKNAME --query 'Stacks[0].Outputs[?OutputKey==`ESSnapshotBucket`].OutputValue' --output text)
$ ES_SNAPSHOT_ROLE_ARN=$(aws cloudformation describe-stacks --stack-name STACKNAME --query 'Stacks[0].Outputs[?OutputKey==`ESSnapshotRole`].OutputValue' --output text)
$ ES_ENDPOINT=$(aws cloudformation describe-stacks --stack-name STACKNAME --query 'Stacks[0].Outputs[?OutputKey==`ESDomainEndpoint`].OutputValue' --output text)
```
* See [python client](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-managedomains-snapshots.html#es-managedomains-snapshot-client-python) for adding an s3 bucket as a repository
```
PUT https://$ES_ENDPOINT/_snapshot/SNAPSHOT_REPO
{
  "type": "s3",
  "settings": {
    "bucket": "$ES_SNAPSHOT_BUCKET",
    "server_side_encryption": true,
    "region": "$AWS_REGION",
    "role_arn": "$ES_SNAPSHOT_ROLE_ARN"
  }
}
```
