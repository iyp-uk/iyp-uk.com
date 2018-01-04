---
title: "Terraform S3 Lambda Elasticsearch"
date: 2018-01-03T10:27:10+01:00
tags: ["Terraform", "AWS", "S3", "Lambda", "Elasticsearch", "S3 notifications"]
---

## Goal

In this post, we will provide the required infrastructure to [index data in Elasticsearch from Events of an S3 bucket with Lambda](https://aws.amazon.com/blogs/database/indexing-metadata-in-amazon-elasticsearch-service-using-aws-lambda-and-python/).

This makes use of [S3 event notification](https://docs.aws.amazon.com/AmazonS3/latest/dev/NotificationHowTo.html) to fire a Lambda function that will index data from the documents added to the S3 bucket in Elasticsearch.

## Elements in use

![Elements in use](https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2016/12/20/Services.png)

## S3 bucket

`main.tf`:

```
provider "aws" {
  shared_credentials_file = "${var.aws_shared_credentials_file}"
  profile                 = "${var.aws_profile}"
  region                  = "${var.aws_region}"
}

resource "aws_s3_bucket" "bucket" {
  count   = 3
  bucket  = "${var.aws_s3_bucket_prefix}-${element(var.aws_s3_buckets, count.index)}"
  acl     = "private"

  tags {
    Name        = "Project's ${element(var.aws_s3_buckets, count.index)} bucket"
    Environment = "${element(var.aws_s3_buckets, count.index)}"
  }
}
```

`variables.tf`:

```
variable "aws_region" {
  default = "eu-west-1"
}
variable "aws_shared_credentials_file" {
  default = "~/.aws/credentials"
}
variable "aws_profile" {
  default = "profile_name"
}
variable "aws_s3_bucket_prefix" {
  default = "some-prefix"
}
variable "aws_s3_buckets" {
  description = "List of buckets to hold data"
  type        = "list"
  default     = ["dev", "uat", "prod"]
}
```

## Lambda with S3 notifications

When Lambda resources are being processed, they need to have:

* A zip file attached (that's the method we will use, it can be a dummy function)
* `s3_*` properties defined when the lambda source code is hosted in an S3 bucket.

Notice then that we're grabbing a file named `lambda_function.py`, in `${path.module}/../lambda/src` where `${path.module}` is where terraform `.tf` files live.

`lambda_function.py`:

```python
def lambda_handler(event, context):
  print "Hello, world!"
```

> `lambda_function.py` and `lambda_handler` method correspond to what is defined below under `aws_lambda_function`'s handler.

`main.tf`:

```
data "archive_file" "lambda_zip_file" {
  type        = "zip"
  source_dir  = "${path.module}/../lambda/src"
  output_path = "${path.module}/../lambda/lambda.zip"
}

resource "aws_iam_role" "iam_lambda_role" {
  name = "iam_lambda_role"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Effect": "Allow"
    }
  ]
}
EOF
}

resource "aws_lambda_permission" "allow_bucket" {
  count         = "${length(var.aws_s3_buckets)}"
  statement_id  = "AllowExecutionFromS3Bucket"
  action        = "lambda:InvokeFunction"
  function_name = "${element(aws_lambda_function.func.*.arn, count.index)}"
  principal     = "s3.amazonaws.com"
  source_arn    = "${element(aws_s3_bucket.bucket.*.arn, count.index)}"
}

resource "aws_lambda_function" "func" {
  filename         = "${path.module}/../lambda/lambda.zip"
  count            = "${length(var.aws_s3_buckets)}"
  function_name    = "drivers_stats_producer_${element(var.aws_s3_buckets, count.index)}"
  role             = "${aws_iam_role.iam_lambda_role.arn}"
  handler          = "lambda_function.lambda_handler"
  runtime          = "python2.7"

  environment {
    variables = {
      env = "${element(var.aws_s3_buckets, count.index)}"
    }
  }
}

resource "aws_s3_bucket_notification" "bucket_notification" {
  count  = "${length(var.aws_s3_buckets)}"
  bucket = "${element(aws_s3_bucket.bucket.*.id, count.index)}"

  lambda_function {
    lambda_function_arn = "${element(aws_lambda_function.func.*.arn, count.index)}"
    events              = ["s3:ObjectCreated:*"]
  }
}
```

## Elasticsearch cluster

Creating an Elasticsearch cluster in AWS takes a while, whether from the UI or with Terraform. 
Be prepared to wait for about 20 minutes or more. 

> Take note that the policy defines the ES instance to be publicly accessible to anyone. 
This is what is shown in the AWS document, but you will want to change that to your needs, it is **not recommended** to keep it like that. 
You can restrict access policy and filter by IP or IAM users, etc. 

```
resource "aws_elasticsearch_domain" "es_domain" {
  domain_name           = "${var.aws_es_domain_name}"
  elasticsearch_version = "${var.aws_es_version}"

  cluster_config {
    instance_type   = "m4.large.elasticsearch"
    instance_count  = 2
  }

  snapshot_options {
    automated_snapshot_start_hour = 23
  }

  ebs_options {
    ebs_enabled = true
    volume_size = 30
  }
}

resource "aws_elasticsearch_domain_policy" "main" {
  domain_name = "${aws_elasticsearch_domain.es_domain.domain_name}"

  access_policies = <<POLICIES
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": "es:*",
      "Resource": "${aws_elasticsearch_domain.es_domain.arn}/*"
    }
  ]
}
POLICIES
}
```
