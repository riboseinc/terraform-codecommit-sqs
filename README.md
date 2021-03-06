# Terraform Module For CodeCommit + SQS

CodeCommit repos created using this terraform module is compatible with
the
[Jenkins AWS CodeCommit Trigger Plugin](https://github.com/riboseinc/aws-codecommit-trigger-plugin),
i.e., changes to the git repo automatically triggers the plugin.

This module is available on the [Terraform Registry](https://registry.terraform.io/modules/riboseinc/codecommit-sqs/aws/).

## Sample Usage

```go
variable "aws-account-id" {
  default = "my-aws-account-id"
}

variable "aws-region" {
  default = "my-aws-region"
}

variable "sns-topic-prefix" {
  default = "codecommit-"
}

variable "sns-topic-suffix" {
  default = "-topic"
}

provider "aws" {
  region = "${var.aws-region}"
  alias = "default"
}

resource "aws_sqs_queue" "main" {
  name = "codecommit-notifications-queue"
  delay_seconds = 90
  max_message_size = 2048
  message_retention_seconds = 86400
  receive_wait_time_seconds = 10
}

resource "aws_sqs_queue_policy" "sns" {
  queue_url = "${aws_sqs_queue.main.id}"
  policy = "${data.aws_iam_policy_document.sns-sqs-policy.json}"
}

data "aws_iam_policy_document" "sns-sqs-policy" {
  policy_id = "arn:aws:sqs:us-east-1:${var.aws-account-id}:testing/SQSDefaultPolicy"

  statement {
    sid = "SubscribeToSNS"
    effect = "Allow"
    principals {
      type = "AWS"
      identifiers = [ "*" ]
    }
    actions = [ "SQS:SendMessage" ]
    resources = [ "${aws_sqs_queue.main.arn}" ]
    condition {
      test = "ArnLike"
      variable = "aws:SourceArn"
      values = [ "arn:aws:sns:us-east-1:${var.aws-account-id}:${var.sns-topic-prefix}*${var.sns-topic-suffix}" ]
    }
  }
}

module "cc-example_repo" {
  source = "riboseinc/codecommit-sqs/aws"
  reponame = "example-repo"
  aws-account-id = "${var.aws-account-id}"
  sqs-arn = "${aws_sqs_queue.main.arn}"
  sqs-id = "${aws_sqs_queue.main.id}"
  topic-prefix = "${var.sns-topic-prefix}"
  topic-suffix = "${var.sns-topic-suffix}"
  # email-sns-arn = "${aws_sns_topic.codecommit-email.arn}"

  providers = {
    aws = "aws.default"
  }
}

output "cc-example_repo-cc-arn" {
  value = "${module.cc-example_repo.cc-arn}"
}
output "cc-example_repo-sns-name" {
  value = "${module.cc-example_repo.sns-name}"
}
output "cc-example_repo-sns-arn" {
  value = "${module.cc-example_repo.sns-arn}"
}
```

## Enabling With An Email Notification

Some people prefer receiving an email on every commit.

This is how you set it up.

```go
resource "aws_sns_topic" "codecommit-email" {
  name = "codecommit-email-notifications"
  display_name = "CodeCommit notifications"
}

resource "aws_sns_topic_policy" "codecommit-email-sns-policy" {
  arn = "${aws_sns_topic.codecommit-email.arn}"
  policy = "${data.aws_iam_policy_document.codecommit-email-sns-policy.json}"
}

data "aws_iam_policy_document" "codecommit-email-sns-policy" {
  statement {
    sid = "AllowSubscription"
    effect = "Allow"
    principals {
      type = "AWS"
      identifiers = [ "*" ]
    }
    actions = [
      "SNS:Publish",
      "SNS:RemovePermission",
      "SNS:SetTopicAttributes",
      "SNS:DeleteTopic",
      "SNS:ListSubscriptionsByTopic",
      "SNS:GetTopicAttributes",
      "SNS:Receive",
      "SNS:AddPermission",
      "SNS:Subscribe"
    ]
    resources = [ "${aws_sns_topic.codecommit-email.arn}" ]
    condition {
      test = "StringEquals"
      variable = "AWS:SourceOwner"
      values = [ "${var.aws-account-id}" ]
    }
  }

}

output "email-sns-arn" {
  value = "${aws_sns_topic.codecommit-email.arn}"
}

output "email-sns-name" {
  value = "${aws_sns_topic.codecommit-email.name}"
}

# Link it with this module
module "cc-example_repo" {
  source = "riboseinc/codecommit-sqs/aws"
  reponame = "example-repo"
  aws-account-id = "${var.aws-account-id}"
  email-sns-arn = "${aws_sns_topic.codecommit-email.arn}"
  topic-prefix = "${var.sns-topic-prefix}"
  topic-suffix = "${var.sns-topic-suffix}"
  sqs-arn = "${aws_sqs_queue.main.arn}"
  sqs-id = "${aws_sqs_queue.main.id}"

  providers = {
    aws = "aws.default"
  }
}
```
