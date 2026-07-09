# 🏗️ Terraform AWS Learning Modules

> **Production-style, scenario-driven HTML learning modules** for provisioning and connecting AWS services using the [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest).

[![Terraform](https://img.shields.io/badge/Terraform-%3E%3D1.5-7B42BC?logo=terraform&logoColor=white)](https://www.terraform.io/)
[![AWS Provider](https://img.shields.io/badge/AWS_Provider-~%3E6.0-FF9900?logo=amazon-aws&logoColor=white)](https://registry.terraform.io/providers/hashicorp/aws/latest)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![Modules](https://img.shields.io/badge/Modules-4-blue)](#-modules-at-a-glance)

---

## 📖 Overview

This repository contains **four self-contained HTML learning modules** targeting senior engineers and architects who want realistic, production-grade Terraform patterns — not toy examples.

Each module is a **complete, standalone HTML document** (no build tools, no dependencies, no internet connection required) that teaches how to wire multiple AWS services into a coherent, deployable workflow using Terraform HCL.

All Terraform code is validated against **AWS Provider v6.53.0** (July 2026) and uses only current, non-deprecated resource arguments.

---

## 🗂️ Modules at a Glance

| # | Module | Services | Pattern |
|---|--------|----------|---------|
| 1 | [S3 → EventBridge → Lambda → DynamoDB → SNS](#module-1--s3--eventbridge--lambda--dynamodb--sns) | S3, EventBridge, Lambda, DynamoDB, SNS, CloudWatch, IAM | Event-driven ingestion pipeline |
| 2 | [API-Triggered Step Functions Orchestration](#module-2--api-triggered-step-functions-orchestration) | Lambda, Step Functions, DynamoDB, SNS, CloudWatch, IAM | Multi-step durable workflow |
| 3 | [Scheduled Data Processing](#module-3--scheduled-data-processing) | EventBridge, Lambda, S3, DynamoDB, SNS, CloudWatch, IAM | Cron-driven batch processing |
| 4 | [Error Monitoring and Alarms](#module-4--error-monitoring-and-alarms) | CloudWatch Logs, Metric Alarms, Composite Alarms, SNS, IAM | Full observability layer |

---

## 🚀 Quick Start

No installation required. Simply open any HTML file in your browser:

```bash
# Clone the repository
git clone https://github.com/your-username/terraform-aws-learning-modules.git
cd terraform-aws-learning-modules

# Open any module directly in your default browser
open module1.html        # macOS
xdg-open module1.html   # Linux
start module1.html       # Windows
```

> Each HTML file is **100% self-contained** — all styles are embedded inline. No Node.js, no npm, no build step.

---

## 📚 Module Details

### Module 1 · S3 → EventBridge → Lambda → DynamoDB → SNS

**File:** `module1.html` &nbsp;|&nbsp; **825 lines** &nbsp;|&nbsp; **19 AWS resources**

The foundational event-driven ingestion pipeline. A file upload to S3 triggers the entire downstream chain automatically.

```
S3 Bucket ──► EventBridge Rule ──► Lambda Function ──► DynamoDB Table
                                                    └──► SNS Topic
```

**What you'll learn:**
- Using `aws_s3_bucket_notification` with `eventbridge = true` — the modern decoupled pattern (no direct S3→Lambda trigger wiring)
- Why `aws_lambda_permission` is required alongside every `aws_cloudwatch_event_target`
- Scoping EventBridge `event_pattern` to a specific bucket ARN to prevent cross-bucket misfires
- `state = "ENABLED"` on `aws_cloudwatch_event_rule` (replaces the deprecated `is_enabled` argument)
- DynamoDB `PAY_PER_REQUEST` billing with TTL and Point-in-Time Recovery
- Pre-creating CloudWatch log groups to enforce retention from the first Lambda invocation

**Key resources:**
`aws_s3_bucket` · `aws_s3_bucket_notification` · `aws_s3_bucket_public_access_block` · `aws_cloudwatch_event_rule` · `aws_cloudwatch_event_target` · `aws_lambda_function` · `aws_lambda_permission` · `aws_dynamodb_table` · `aws_sns_topic` · `aws_sns_topic_policy` · `aws_iam_role` · `aws_iam_policy` · `aws_cloudwatch_log_group` · `aws_cloudwatch_metric_alarm`

---

### Module 2 · API-Triggered Step Functions Orchestration

**File:** `module2.html` &nbsp;|&nbsp; **992 lines** &nbsp;|&nbsp; **22 AWS resources**

Multi-step durable workflow orchestration. An external caller triggers a dispatcher Lambda that starts a Step Functions Standard Workflow, which sequentially invokes worker Lambdas to write to DynamoDB and publish to SNS.

```
External Caller ──► Dispatcher Lambda ──► Step Functions State Machine
                                               ├──► write_record Lambda ──► DynamoDB
                                               └──► send_notify Lambda  ──► SNS
```

**What you'll learn:**
- Defining three distinct IAM roles: dispatcher Lambda, worker Lambda, and Step Functions execution role — each with least-privilege policies
- Writing Amazon States Language (ASL) definitions using `jsonencode()` — no manual JSON escaping
- `Retry` and `Catch` blocks on every `Task` state for production resilience
- `logging_configuration` on `aws_sfn_state_machine` — including the required `:*` suffix on `log_destination`
- `STANDARD` vs `EXPRESS` workflow trade-offs (exactly-once vs at-least-once, audit trail, duration limits)
- `ResultPath` semantics for merging task output into the state machine's running input

**Key resources:**
`aws_sfn_state_machine` · `aws_lambda_function` (×3) · `aws_dynamodb_table` · `aws_sns_topic` · `aws_iam_role` (×3) · `aws_iam_policy` (×3) · `aws_cloudwatch_log_group` (×4) · `aws_cloudwatch_metric_alarm` (×2)

---

### Module 3 · Scheduled Data Processing

**File:** `module3.html` &nbsp;|&nbsp; **960 lines** &nbsp;|&nbsp; **21 AWS resources**

Cron-driven batch processing pipeline. EventBridge fires on a schedule, Lambda reads and aggregates CSV files from S3, writes a summary to DynamoDB, and publishes completion or failure notifications to SNS.

```
EventBridge Schedule ──► Lambda (batch_processor)
                              ├── reads ──► S3 Bucket (date-partitioned prefix)
                              ├── writes ──► DynamoDB (job_results)
                              └── publishes ──► SNS Topic
                                                    ▲
                                         CloudWatch Alarms (errors, throttles,
                                         duration, missed schedule)
```

**What you'll learn:**
- EventBridge **6-field cron syntax** (vs Unix 5-field) — with a full reference table of common expressions
- Using a Terraform variable for `schedule_expression` to run `rate(5 minutes)` in dev and `cron(0 2 * * ? *)` in production
- Four CloudWatch alarms including the critical **"missed schedule" alarm** (`treat_missing_data = "breaching"`)
- S3 lifecycle rules for automatic data tiering and expiration
- Idempotency design for at-least-once Lambda delivery using DynamoDB conditional writes
- Lambda memory/timeout sizing guidance for I/O-bound vs CPU-bound workloads

**Key resources:**
`aws_cloudwatch_event_rule` (schedule) · `aws_cloudwatch_event_target` · `aws_lambda_function` · `aws_s3_bucket` · `aws_s3_bucket_lifecycle_configuration` · `aws_dynamodb_table` · `aws_sns_topic` · `aws_iam_role` · `aws_iam_policy` · `aws_cloudwatch_log_group` · `aws_cloudwatch_metric_alarm` (×4)

---

### Module 4 · Error Monitoring and Alarms

**File:** `module4.html` &nbsp;|&nbsp; **1,249 lines** &nbsp;|&nbsp; **28 AWS resources**

A complete, standalone observability layer that can be applied on top of any of the previous modules. Covers the full CloudWatch stack from log groups to composite alarms to dashboards.

```
AWS Service Metrics ──┐
                      ├──► CloudWatch Metric Alarms ──┐
Lambda Log Groups ────┤                               ├──► Composite Alarm ──► SNS (critical)
  └─ Metric Filters ──┘                               │
                                                      └──► SNS (warning)
CloudWatch Dashboard (single-pane-of-glass view)
CloudWatch Logs Insights (saved queries)
```

**What you'll learn:**
- `aws_cloudwatch_log_metric_filter` — extracting custom metrics from Lambda logs using literal, word, and JSON field patterns
- Using `for_each` to create per-function alarms across all monitored Lambda functions from a single resource block
- `extended_statistic = "p99"` for latency alarms (mutually exclusive with `statistic`)
- `aws_cloudwatch_composite_alarm` with boolean `alarm_rule` expressions to reduce alert noise
- `aws_cloudwatch_dashboard` with `jsonencode()` for a multi-widget single-pane-of-glass view
- `aws_cloudwatch_query_definition` for saved Logs Insights queries
- Separating critical vs warning SNS topics for tiered on-call routing
- Operational runbook guidance embedded directly in alarm descriptions

**Key resources:**
`aws_sns_topic` (×2) · `aws_sns_topic_policy` (×2) · `aws_cloudwatch_log_group` · `aws_cloudwatch_log_metric_filter` (×3 types) · `aws_cloudwatch_metric_alarm` (×10+) · `aws_cloudwatch_composite_alarm` (×2) · `aws_cloudwatch_dashboard` · `aws_cloudwatch_query_definition` (×2)

---

## 🧩 AWS Resources Covered

| Resource | Module(s) |
|----------|-----------|
| `aws_s3_bucket` | 1, 3 |
| `aws_s3_bucket_notification` | 1 |
| `aws_s3_bucket_public_access_block` | 1, 3 |
| `aws_s3_bucket_versioning` | 1, 3 |
| `aws_s3_bucket_server_side_encryption_configuration` | 1, 3 |
| `aws_s3_bucket_lifecycle_configuration` | 3 |
| `aws_cloudwatch_event_rule` | 1, 3 |
| `aws_cloudwatch_event_target` | 1, 3 |
| `aws_lambda_function` | 1, 2, 3 |
| `aws_lambda_permission` | 1, 2, 3 |
| `aws_dynamodb_table` | 1, 2, 3 |
| `aws_sns_topic` | 1, 2, 3, 4 |
| `aws_sns_topic_policy` | 1, 2, 3, 4 |
| `aws_sns_topic_subscription` | 1, 2, 3, 4 |
| `aws_iam_role` | 1, 2, 3, 4 |
| `aws_iam_policy` | 1, 2, 3, 4 |
| `aws_iam_role_policy_attachment` | 1, 2, 3, 4 |
| `aws_iam_policy_document` (data) | 1, 2, 3, 4 |
| `aws_sfn_state_machine` | 2 |
| `aws_cloudwatch_log_group` | 1, 2, 3, 4 |
| `aws_cloudwatch_log_metric_filter` | 4 |
| `aws_cloudwatch_metric_alarm` | 1, 2, 3, 4 |
| `aws_cloudwatch_composite_alarm` | 4 |
| `aws_cloudwatch_dashboard` | 4 |
| `aws_cloudwatch_query_definition` | 4 |

---

## ⚙️ Prerequisites

To apply the Terraform code from these modules in your own AWS account:

| Requirement | Version |
|-------------|---------|
| [Terraform CLI](https://developer.hashicorp.com/terraform/install) | ≥ 1.5 |
| [AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest) | ~> 6.0 |
| AWS credentials | Any valid method (env vars, profile, instance profile) |
| Python | 3.12 (for Lambda handler examples) |

```hcl
# Minimal provider configuration used across all modules
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

---

## 🔑 Key Design Principles

These modules consistently apply the following production patterns:

- **`default_tags` on the provider block** — tags every resource automatically without repetition
- **`source_code_hash` on Lambda** — ensures redeployment only when code actually changes
- **`aws_iam_policy_document` data sources** — type-safe, readable IAM policy authorship
- **`jsonencode()` for inline JSON** — eliminates manual escaping in EventBridge patterns, ASL definitions, and dashboard bodies
- **`for_each` for repeated resources** — one alarm block covers all Lambda functions
- **`depends_on` for log groups** — prevents Lambda from creating unmanaged log groups without retention policies
- **Least-privilege IAM** — every role has only the specific actions it needs, scoped to specific resource ARNs
- **Environment variables over hardcoding** — Lambda functions receive ARNs and names via `environment.variables`, never hardcoded

---

## ⚠️ Common Pitfalls Reference

A quick-reference table of the most frequent mistakes covered across all modules:

| Pitfall | Module | Fix |
|---------|--------|-----|
| Missing `aws_lambda_permission` for EventBridge | 1, 3 | Always pair `aws_cloudwatch_event_target` → Lambda with `aws_lambda_permission` |
| `acl` argument on `aws_s3_bucket` | 1, 3 | Removed in provider v4+; use `aws_s3_bucket_acl` or omit for private |
| `is_enabled` on EventBridge rule | 1, 3 | Deprecated; use `state = "ENABLED"` |
| 5-field Unix cron in EventBridge | 3 | EventBridge requires 6-field cron: `cron(Min Hr DoM Mon DoW Yr)` |
| Missing `:*` on SFN `log_destination` | 2 | Must be `"${log_group.arn}:*"` — omitting silently drops all logs |
| Wrong trust principal for SFN role | 2 | Must be `states.amazonaws.com`, not `lambda.amazonaws.com` |
| Over-defining DynamoDB `attribute` blocks | 1, 2, 3 | Only declare attributes used as key schema — others cause perpetual diff |
| `statistic` + `extended_statistic` together | 4 | Mutually exclusive — use one or the other per alarm |
| CloudWatch filter pattern using regex | 4 | CloudWatch uses its own pattern syntax, not regex |
| SNS topic policy missing CloudWatch principal | 4 | Add `cloudwatch.amazonaws.com` to `SNS:Publish` statement |

---

## 📁 Repository Structure

```
terraform-aws-learning-modules/
├── README.md          ← You are here
├── module1.html       ← S3 → EventBridge → Lambda → DynamoDB → SNS
├── module2.html       ← API-Triggered Step Functions Orchestration
├── module3.html       ← Scheduled Data Processing
└── module4.html       ← Error Monitoring and Alarms with CloudWatch
```

---

## 🔗 References

- [Terraform AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Amazon States Language Reference](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-amazon-states-language.html)
- [EventBridge Event Patterns](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-patterns.html)
- [EventBridge Schedule Expressions](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-scheduled-rule-pattern.html)
- [CloudWatch Filter Pattern Syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/FilterAndPatternSyntax.html)
- [CloudWatch Composite Alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create_Composite_Alarm.html)
- [Lambda Execution Environment](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)

---

## 📄 License

This project is licensed under the **Apache License 2.0** — see the [LICENSE](LICENSE) file for details.

```
Copyright 2026 Terraform AWS Learning Modules Contributors

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

---

<p align="center">
  Built for engineers who learn by reading real code, not slides.
</p>
