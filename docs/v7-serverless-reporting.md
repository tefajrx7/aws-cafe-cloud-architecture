# V7 — Serverless Reporting (Lambda + SNS)

> Completed as part of AWS Academy Challenge Labs (training environment).

## Business Requirement

The café is growing and the owners want a **daily sales report** by email, to plan ingredient orders and see the effect of promotions. A cron job on the web server worked but slowed the production site, and a dedicated EC2 instance would sit idle almost all day. The decision: move report generation to **AWS Lambda** (pay only while it runs) and deliver the result with **Amazon SNS**.

## Before → After

```mermaid
flowchart LR
  subgraph Before["Before (V6)"]
    W1["Café web server<br/>(public subnet)"] --> D1[("RDS database<br/>(private subnets)")]
  end
  subgraph After["After (V7)"]
    EB["EventBridge rule<br/>daily 14:25 UTC"] --> L1["Lambda<br/>salesAnalysisReport"]
    L1 --> L2["Lambda<br/>salesAnalysisReportDataExtractor<br/>(in VPC, private subnets 1+2)"]
    L2 -->|"MySQL 3306 over TLS"| D2[("RDS database")]
    L1 --> SNS["SNS topic<br/>SalesReportTopic"] --> M["Email subscriber"]
    W2["Café web server<br/>(no reporting load)"] --> D2
  end
  Before --> After
```

### Reference architecture

![Reference: starting architecture from the AWS Academy lab](../diagrams/v7/V7-ref-start-architecture.png)
*Reference: starting architecture from the AWS Academy lab.*

![Reference: target architecture from the AWS Academy lab](../diagrams/v7/V7-ref-target-architecture.png)
*Reference: target architecture from the AWS Academy lab.*

## AWS Services Used

| Service | Role in V7 |
|---|---|
| AWS Lambda | Two functions: a data extractor (inside the VPC) and the report generator |
| Amazon VPC / Security Groups | `LambdaSG` for the extractor; `DatabaseSG` updated to accept it |
| Amazon RDS | Source of the café sales data (reached privately) |
| Amazon SNS | Topic `SalesReportTopic` with an email subscription |
| Amazon EventBridge | Scheduled rule that runs the report daily |
| Amazon CloudWatch Logs | Debugging the failed test runs |
| AWS IAM | Execution roles for the functions and `mySchedulerRole` for the schedule |

## Implementation Highlights

- **Network access for Lambda:** created `LambdaSG` (all outbound) in the Lab VPC and added a second MySQL/Aurora (3306) inbound rule on `DatabaseSG` whose source is `LambdaSG`, so only the function, not the whole network, can reach the database.
- **DataExtractor:** a Lambda attached to Private subnet 1 and 2 with `LambdaSG`, 128 MB memory, 30 s timeout, using the pre-made `salesAnalysisReportDERole`. It must live in the VPC because the database is in private subnets.
- **Report function:** `salesAnalysisReport`, 128 MB, 30 s, role `salesAnalysisReportRole`. It is not placed in the VPC.
- **SNS:** standard topic `SalesReportTopic` (display name *Sales Report Topic*); its ARN is passed to the report function as the `topicARN` environment variable; an email subscription was created and confirmed from the inbox.
- **Schedule:** EventBridge rule `dailySalesReportRule` with a cron expression for 14:25 UTC every day, targeting `salesAnalysisReport` through the existing `mySchedulerRole`.

## Challenges & Decisions

**Problem 1 — first test failed.** Running `salesAnalysisReport` returned *"No body in returned data"*. The report function's own log only repeated that message, so the real cause was upstream.

**Problem 2 — database refused the connection.** The DataExtractor log showed it failed to connect to the database with error 3159: *connections using insecure transport are prohibited while `--require_secure_transport=ON`*. The network path (security groups, subnets) was therefore fine; the database required TLS.

**Fix.** The extractor package was updated with the RDS CA bundle (`global-bundle.pem`) and the connection code now passes it as the SSL certificate. After that the test returned `statusCode 200` and the report email arrived.

**Decisions**
- Security group *source* = `LambdaSG` instead of a CIDR range: least privilege and no dependency on IP addresses.
- Extractor in two private subnets (two AZs); report function left outside the VPC because it only needs to call the extractor and SNS, not the database.
- Lambda instead of cron-on-EC2 or a separate instance: no impact on the web server and no 24/7 cost.

<!-- CONFIRM (from screenshots, not from notes): (a) the TLS fix (global-bundle.pem + ssl code) was my own change; (b) the lab text says Python 3.11 but both functions show Python 3.13 in the console; (c) no screenshot shows the report arriving from the *scheduled* run (the only email shown is from the manual test on 2026-11-02). -->
<!-- TODO: add any other problems I hit and how I solved them -->

## Validation & Evidence

![DatabaseSG now allows MySQL/Aurora (3306) from both the café app SG and LambdaSG](../evidence/v7/V7-E04-databasesg-inbound-two-sources.png)
*V7-E04 — The database now trusts two sources: the café web server and the new Lambda security group.*

![Attaching the function to Lab VPC, Private subnet 1 + 2 and LambdaSG](../evidence/v7/V7-E07-dataextractor-vpc-subnets-sg.png)
*V7-E07 — DataExtractor attached to the Lab VPC, Private subnets 1 and 2, with LambdaSG.*

![DataExtractor log: database connection refused because the server requires TLS](../evidence/v7/V7-E27-dataextractor-logs-tls-error.png)
*V7-E27 — First failure: the database refused a non-TLS connection.*

![DataExtractor code updated to connect over TLS using the RDS CA bundle](../evidence/v7/V7-E28-dataextractor-code-ssl-fix.png)
*V7-E28 — The fix: the extractor now connects over TLS with the RDS CA bundle.*

![Daily Sales Analysis Report email received](../evidence/v7/V7-E31-daily-sales-report-email.png)
*V7-E31 — The daily sales report arriving by email through SNS.*

![Rule created and enabled](../evidence/v7/V7-E35-eventbridge-rule-created.png)
*V7-E35 — EventBridge rule created to run the report every day.*

### Evidence index (all screenshots)

| ID | What it shows | File |
|---|---|---|
| V7-E01 | Creating the LambdaSG security group in the Lab VPC (all outbound allowed) | `V7-E01-create-security-group-lambdasg.png` |
| V7-E02 | LambdaSG created | `V7-E02-lambdasg-created.png` |
| V7-E03 | Editing the database security group to add a second inbound rule | `V7-E03-databasesg-edit-inbound-rules.png` |
| V7-E04 | DatabaseSG now allows MySQL/Aurora (3306) from both the café app SG and LambdaSG | `V7-E04-databasesg-inbound-two-sources.png` |
| V7-E05 | Lambda console before creating the functions | `V7-E05-lambda-console-overview.png` |
| V7-E06 | Creating salesAnalysisReportDataExtractor with the custom execution role | `V7-E06-create-dataextractor-function.png` |
| V7-E07 | Attaching the function to Lab VPC, Private subnet 1 + 2 and LambdaSG | `V7-E07-dataextractor-vpc-subnets-sg.png` |
| V7-E08 | VPC configuration summary (2 subnets, 1 security group) | `V7-E08-dataextractor-vpc-summary.png` |
| V7-E09 | Function code package uploaded | `V7-E09-dataextractor-code-uploaded.png` |
| V7-E10 | Basic settings: 128 MB memory, 30 s timeout, salesAnalysisReportDERole | `V7-E10-dataextractor-basic-settings.png` |
| V7-E11 | General configuration confirmed | `V7-E11-dataextractor-general-config.png` |
| V7-E12 | Runtime and handler settings | `V7-E12-dataextractor-runtime-handler.png` |
| V7-E13 | Handler code in the console editor | `V7-E13-dataextractor-code-view.png` |
| V7-E14 | Report function code uploaded | `V7-E14-salesanalysisreport-code-uploaded.png` |
| V7-E15 | Runtime, handler and code properties | `V7-E15-salesanalysisreport-runtime-handler.png` |
| V7-E16 | Basic settings: 128 MB, 30 s, salesAnalysisReportRole | `V7-E16-salesanalysisreport-basic-settings.png` |
| V7-E17 | General configuration confirmed | `V7-E17-salesanalysisreport-general-config.png` |
| V7-E18 | Creating the standard SNS topic SalesReportTopic | `V7-E18-create-sns-topic.png` |
| V7-E19 | Topic created, no subscriptions yet | `V7-E19-sns-topic-created.png` |
| V7-E20 | topicARN environment variable set on salesAnalysisReport | `V7-E20-topicarn-environment-variable.png` |
| V7-E21 | Creating the email subscription on the topic | `V7-E21-create-email-subscription.png` |
| V7-E22 | Subscription created, status Pending confirmation | `V7-E22-subscription-pending-confirmation.png` |
| V7-E23 | AWS confirmation email received | `V7-E23-confirmation-email-received.png` |
| V7-E24 | Subscription confirmed | `V7-E24-subscription-confirmed.png` |
| V7-E25 | First test run of salesAnalysisReport failed ("No body in returned data") | `V7-E25-first-test-failed.png` |
| V7-E26 | CloudWatch log of salesAnalysisReport showing the same error | `V7-E26-report-logs-no-body-error.png` |
| V7-E27 | DataExtractor log: database connection refused because the server requires TLS | `V7-E27-dataextractor-logs-tls-error.png` |
| V7-E28 | DataExtractor code updated to connect over TLS using the RDS CA bundle | `V7-E28-dataextractor-code-ssl-fix.png` |
| V7-E29 | Second test run succeeded (statusCode 200, report sent) | `V7-E29-test-succeeded.png` |
| V7-E30 | CloudWatch log of the successful run | `V7-E30-cloudwatch-success-log.png` |
| V7-E31 | Daily Sales Analysis Report email received | `V7-E31-daily-sales-report-email.png` |
| V7-E32 | Defining the scheduled rule dailySalesReportRule | `V7-E32-eventbridge-rule-detail.png` |
| V7-E33 | Cron schedule, 14:25 UTC every day | `V7-E33-eventbridge-cron-schedule.png` |
| V7-E34 | Target: salesAnalysisReport Lambda with mySchedulerRole | `V7-E34-eventbridge-target-lambda.png` |
| V7-E35 | Rule created and enabled | `V7-E35-eventbridge-rule-created.png` |
| V7-E36 | Rule targets tab confirming the Lambda target | `V7-E36-eventbridge-rule-targets.png` |

<details><summary><b>More evidence (all screenshots, in lab order)</b></summary>

#### Task 2 — Security groups

![Creating the LambdaSG security group in the Lab VPC (all outbound allowed)](../evidence/v7/V7-E01-create-security-group-lambdasg.png)
*V7-E01 — Creating the LambdaSG security group in the Lab VPC (all outbound allowed)*

![LambdaSG created](../evidence/v7/V7-E02-lambdasg-created.png)
*V7-E02 — LambdaSG created*

![Editing the database security group to add a second inbound rule](../evidence/v7/V7-E03-databasesg-edit-inbound-rules.png)
*V7-E03 — Editing the database security group to add a second inbound rule*

![DatabaseSG now allows MySQL/Aurora (3306) from both the café app SG and LambdaSG](../evidence/v7/V7-E04-databasesg-inbound-two-sources.png)
*V7-E04 — DatabaseSG now allows MySQL/Aurora (3306) from both the café app SG and LambdaSG*


#### Task 2 — DataExtractor function

![Lambda console before creating the functions](../evidence/v7/V7-E05-lambda-console-overview.png)
*V7-E05 — Lambda console before creating the functions*

![Creating salesAnalysisReportDataExtractor with the custom execution role](../evidence/v7/V7-E06-create-dataextractor-function.png)
*V7-E06 — Creating salesAnalysisReportDataExtractor with the custom execution role*

![Attaching the function to Lab VPC, Private subnet 1 + 2 and LambdaSG](../evidence/v7/V7-E07-dataextractor-vpc-subnets-sg.png)
*V7-E07 — Attaching the function to Lab VPC, Private subnet 1 + 2 and LambdaSG*

![VPC configuration summary (2 subnets, 1 security group)](../evidence/v7/V7-E08-dataextractor-vpc-summary.png)
*V7-E08 — VPC configuration summary (2 subnets, 1 security group)*

![Function code package uploaded](../evidence/v7/V7-E09-dataextractor-code-uploaded.png)
*V7-E09 — Function code package uploaded*

![Basic settings: 128 MB memory, 30 s timeout, salesAnalysisReportDERole](../evidence/v7/V7-E10-dataextractor-basic-settings.png)
*V7-E10 — Basic settings: 128 MB memory, 30 s timeout, salesAnalysisReportDERole*

![General configuration confirmed](../evidence/v7/V7-E11-dataextractor-general-config.png)
*V7-E11 — General configuration confirmed*

![Runtime and handler settings](../evidence/v7/V7-E12-dataextractor-runtime-handler.png)
*V7-E12 — Runtime and handler settings*

![Handler code in the console editor](../evidence/v7/V7-E13-dataextractor-code-view.png)
*V7-E13 — Handler code in the console editor*


#### Task 3 — salesAnalysisReport function

![Report function code uploaded](../evidence/v7/V7-E14-salesanalysisreport-code-uploaded.png)
*V7-E14 — Report function code uploaded*

![Runtime, handler and code properties](../evidence/v7/V7-E15-salesanalysisreport-runtime-handler.png)
*V7-E15 — Runtime, handler and code properties*

![Basic settings: 128 MB, 30 s, salesAnalysisReportRole](../evidence/v7/V7-E16-salesanalysisreport-basic-settings.png)
*V7-E16 — Basic settings: 128 MB, 30 s, salesAnalysisReportRole*

![General configuration confirmed](../evidence/v7/V7-E17-salesanalysisreport-general-config.png)
*V7-E17 — General configuration confirmed*


#### Task 4 — SNS topic

![Creating the standard SNS topic SalesReportTopic](../evidence/v7/V7-E18-create-sns-topic.png)
*V7-E18 — Creating the standard SNS topic SalesReportTopic*

![Topic created, no subscriptions yet](../evidence/v7/V7-E19-sns-topic-created.png)
*V7-E19 — Topic created, no subscriptions yet*

![topicARN environment variable set on salesAnalysisReport](../evidence/v7/V7-E20-topicarn-environment-variable.png)
*V7-E20 — topicARN environment variable set on salesAnalysisReport*


#### Task 5 — Email subscription

![Creating the email subscription on the topic](../evidence/v7/V7-E21-create-email-subscription.png)
*V7-E21 — Creating the email subscription on the topic*

![Subscription created, status Pending confirmation](../evidence/v7/V7-E22-subscription-pending-confirmation.png)
*V7-E22 — Subscription created, status Pending confirmation*

![AWS confirmation email received](../evidence/v7/V7-E23-confirmation-email-received.png)
*V7-E23 — AWS confirmation email received*

![Subscription confirmed](../evidence/v7/V7-E24-subscription-confirmed.png)
*V7-E24 — Subscription confirmed*


#### Task 6 — Testing

![First test run of salesAnalysisReport failed ("No body in returned data")](../evidence/v7/V7-E25-first-test-failed.png)
*V7-E25 — First test run of salesAnalysisReport failed ("No body in returned data")*

![CloudWatch log of salesAnalysisReport showing the same error](../evidence/v7/V7-E26-report-logs-no-body-error.png)
*V7-E26 — CloudWatch log of salesAnalysisReport showing the same error*

![DataExtractor log: database connection refused because the server requires TLS](../evidence/v7/V7-E27-dataextractor-logs-tls-error.png)
*V7-E27 — DataExtractor log: database connection refused because the server requires TLS*

![DataExtractor code updated to connect over TLS using the RDS CA bundle](../evidence/v7/V7-E28-dataextractor-code-ssl-fix.png)
*V7-E28 — DataExtractor code updated to connect over TLS using the RDS CA bundle*

![Second test run succeeded (statusCode 200, report sent)](../evidence/v7/V7-E29-test-succeeded.png)
*V7-E29 — Second test run succeeded (statusCode 200, report sent)*

![CloudWatch log of the successful run](../evidence/v7/V7-E30-cloudwatch-success-log.png)
*V7-E30 — CloudWatch log of the successful run*

![Daily Sales Analysis Report email received](../evidence/v7/V7-E31-daily-sales-report-email.png)
*V7-E31 — Daily Sales Analysis Report email received*


#### Task 7 — EventBridge

![Defining the scheduled rule dailySalesReportRule](../evidence/v7/V7-E32-eventbridge-rule-detail.png)
*V7-E32 — Defining the scheduled rule dailySalesReportRule*

![Cron schedule, 14:25 UTC every day](../evidence/v7/V7-E33-eventbridge-cron-schedule.png)
*V7-E33 — Cron schedule, 14:25 UTC every day*

![Target: salesAnalysisReport Lambda with mySchedulerRole](../evidence/v7/V7-E34-eventbridge-target-lambda.png)
*V7-E34 — Target: salesAnalysisReport Lambda with mySchedulerRole*

![Rule created and enabled](../evidence/v7/V7-E35-eventbridge-rule-created.png)
*V7-E35 — Rule created and enabled*

![Rule targets tab confirming the Lambda target](../evidence/v7/V7-E36-eventbridge-rule-targets.png)
*V7-E36 — Rule targets tab confirming the Lambda target*


</details>

## Outcome

Sales reporting no longer touches the production web server: a scheduled Lambda pulls the data from RDS through a VPC-attached extractor and emails the report through SNS, and the cost is only the few seconds it runs each day.

---
**Previous:** [V6 — Infrastructure Automation](v6-iac-automation.md)
