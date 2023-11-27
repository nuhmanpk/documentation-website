---
layout: default
title: dynamodb
parent: Sources
grand_parent: Pipelines
nav_order: 3
---

# dynamodb

The `dynamodb` source enables change data capture (CDC) on [DynamoDB](https://aws.amazon.com/dynamodb/) tables. It can receive table events such as `create`, `update`, or `delete` via DynamoDB streams and supports initial snapshots using [point-in-time recovery (PITR)](https://aws.amazon.com/dynamodb/pitr/).

The source includes two ingestion options to stream DynamoDB events:

1. _Full Initial Snapshot_ using [point-in-time recovery (PITR)](https://aws.amazon.com/dynamodb/pitr/) gets an initial snapshot of the current state of the DynamoDB table. This requires the PITR Snapshots and DyanmoDB option enabled on your DynamoDB table.
2.  Stream events from DynamoDB streams without full initial snapshots. This is useful for users who already have a mechanism for snapshots within their pipelines. This requires that DynamoDB stream is enabled on the DynamoDB table.

## Usage

The following example pipeline specifies DynamoDB as a source. It ingests data from a DyanmoDB table, named `table-a` through a PITR snapshot. It also indicates the `start_position`, which tells the pipeline how to read events DynamoDB stream. 

```json
version: "2"
cdc-pipeline:
  source:
    dynamodb:
      tables:
        - table_arn: "arn:aws:dynamodb:us-west-2:123456789012:table/table-a"
          export:
            s3_bucket: "test-bucket"
            s3_prefix: "xxx/"
          stream:
            start_position: "LATEST" # Read latest data from streams (Default)
      aws:
        region: "us-west-2"
        sts_role_arn: "role-arn"
```

## Configuration options

The following tables describe the configuration options for the `dynamodb` source.


The following settings are shared between the `dynamodb` source.

Option | Required | Type | Description
:--- | :--- | :--- | :---
`aws` | Yes | AWS | The AWS configuration. See [aws](#aws) for more information.
`acknowledgments` | No | Boolean  | When `true`, enables `s3` sources to receive [end-to-end acknowledgments]({{site.url}}{{site.baseurl}}/data-prepper/pipelines/pipelines#end-to-end-acknowledgments) when events are received by OpenSearch sinks.
`tables` | Yes | Object | The configuration for the DynamoDB table. See [tables](#tables) for more information.

### aws

Use the following options in the AWS configuration:

Option | Required | Type | Description
:--- | :--- | :--- | :---
`region` | No | String | The AWS Region to use for credentials. Defaults to [standard SDK behavior to determine the Region](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/region-selection.html).
`sts_role_arn` | No | String | The AWS Security Token Service (AWS STS) role to assume for requests to Amazon SQS and Amazon S3. Defaults to `null`, which will use the [standard SDK behavior for credentials](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/credentials.html).
`aws_sts_header_overrides` | No | Map | A map of header overrides that the IAM role assumes for the sink plugin.


### tables

Use the following options with the `tables` configuration.

Option | Required | Type | Description
:--- | :--- | :--- | :---
`s3_bucket` | Yes | String | The destination bucket that stores the exported data files.
`s3_prefix` | No | String | The custom prefix for the S3 bucket.
`s3_sse_kms_key_id` | No | String |  An AWS KMS Customer Managed Key (CMK) that encrypts the export data files. The `key_id` is the ARN of the KMS key, for example, `arn:aws:kms:us-west-2:123456789012:key/0a4bc22f-bb96-4ad4-80ca-63b12b3ec147`.
`s3_region` | No | String | The region for the S3 bucket.
`start_position` | No | String | The start position from when the source reads events from the stream when the DynamoDB stream option is enabled on the table `LATEST` starts reading event about the most recent stream record. 






