# Domain 4: Monitoring and Logging

## 4.1 CloudWatch Metrics

Provides metrics for EVERY services in AWS

Metrics is a variable to monitor (CPUUtilization, NetworkIn)

Metrics belong to namespaces

Dimension is an attribute of a metric (instance id, environemnt, etc..)

up to 40 Dimensions per metric

Metric have timestamps

Can create CloudWatch dashboard of metrics

Can create own CloudWatch Custom Metrics (for RAM for example)

1. CloudWatch Metric Streams

- continually stream CloudWatch metrics to a destination of your choice, with *near-real-time* delivery and low latency

    - Amazon Kinesis Data Firehose (and the its destinations)

    - 3rd Party service provider: Datadog, Dynatrace, new Relic, Splunk, Sumo Logic..

[CW Metrics] --- streamed at near real time ---> [Kinesis Firehose] 
    
    ---> [S3] ---> [Athena] to analyze

    or

    ---> [Amazon Redshift]

    or

    ---> Amazon OpenSearch

- Option to filter metrics to only stream a subset of them

2. Custom Metrics

Possible to define and send your own custom metrics to CloudWatch

- Example: memory (RAM) usage, disk space, number of loggen in users

- Use API call `PutMetricData`

- Ability to use dimensions (attributes) to segment metrics:

    - instance.id

    - environment.name

- Metric resolution (StorageResolution API parameter - two possible value):

    - Standard: 1 minute (60 seconds)

    - High Resolution: 1/5/10/30 second(s) - Higher cost

- IMPORTANT: Accept metrics data points two weeks in the past and two hours in the future (make sure to configure your EC2 instance time correctly)

### CloudWatch Anomaly Detection

Continuously analyze metrics to determine normal baselines and surface anomalies using ML algorithms

It creates a model of the metric's expected values (based on metric's past data)

show you which values in the graph are out of the normal range

Allows you to create Alarms based on metric's expected value (instead of static Threshold)

Ability to exclude specified time periods or events from being trained

## 4.2 Amazon Lookout Metrics

Automatically detect anomalies within metrics and identify their root causes using ML

Detects and diagnoses errors within your data with no amnual intervension

Integrates with different AWS Services and 3rd party Saas apps through *AppFlow*

Send alerts to SNS, Lambda, Slack, Webhooks

[Metrics data may come from s3/Redshift/CloudWatch/RDS] --- connect your data through supported data source integrations [App Flow: salesForce/Marketo/Zendesk] ---> [Amazon Lookout] (continually monitor metrics using ML to detect abnormal changes and identify root causes) --- alerts / custom actions can be sent to ---> SNS or Lambda or vosualize results in AWS Console or download using APIs

## 4.3 CloudWatch Logs

A place to store your logs in AWS.

- Log Groups: arbitrary name, usually representing an application

    - in Log Groups we have Log Stream: instances within an application / log files / contianers

- can define log expiration policies (never expire. 1 day, up to 10 years)

- CloudWatch Logs can send logs to:

    - S3 (exports)

    - Kinesis Data Stream

    - Kinesis Data Firehose

    - AWS lambda

    - OpenSearch

- Logs are encrypted by default

- can setup KMS-based encryption with your own keys

1. Sources:

- SDK, CloudWatch Logs Agent, CloudWatch Unified Agent

- Elastic Beanstalk: collections of logs from application

- ECS: collection from containers

- Lambda: collection from funtrion logs

- VPC Flow Logs: VPC Specific logs

- API Gateway

- CloudTrail based on filter

- Route53: Log DNS queries

2. Querying logs from CloudWatch Logs: use * CloudWatch Logs Insights*

    - Allow you to search and analyze log data stored in CloudWatch losg

    - Example: find specific Ip inside a log, count occurences of "ERROR" in your logs

    - Provides a purpose-built query language

        - Automatically discovers field from AWS services and JSON log events

        - fetch desired event field, filter based on conditions, calculate aggregate statistics, soft event, limit number of events...

        - can save queries and add them to CloudWatch dashboard

    - Can query multiple Log Gorups in different AWS accounts

    - its a query engine, not a real-time engine

3. S3 Exports

- Log data can take up to 12 Hours to become available for export

- Use API call `CreateExportTask`

- NOT near real time or real time.. or Logs Subscriptions instead

4. CloudWatch Logs Subscription

- Get a real time events from CloudWatch logs for processing and analysis

- Send to Kinesis Data Streams, Kinesis Data Firehose, or Lambda

- *Subscription Filter* - filter which logs are events delivered to your destination

[CloudWatch logs] --- logs ---> [subscription filter] 

    ---> Kinesis Data Streams 

        ---> Kinesis Data Firehose, Kinesis Data Analytics, EC2, Lambda

    ---> Kinesis Data Firehose

        ---> near real time to S3

        ---> OpenSearch Service

    ---> Lambda

        ---> send data in real time to OpenSearch Service

5. CloudWatch Logs Aggregation Multi-Account & Multi Region

- we have Account A in region 1, Account B in region 2, Account 3 in Region 3

- each account has [CloudWatch Logs] ---> [Subscription filter]

- All subscription filters send to 1 Kinesis Data Streams

- Kinesis data stream sens to Kinesis Data Firehose --- near realtime ---> S3

6. Cross Account subscription:

    - send log events to resources ina diferent AWS accounts (KDS, KDF)

    - Sender account: [cloudWatch logs] --- logs ---> Subscription Filter

    - Sub filter of Sender account sends the logs to [Subscription Destination] of recipient account.

    - Need to attach *Destination Access Policy*

    - Create IAM role in Subscription Destination recipient to allow *Put Record* to KDS

        - this role must be able to be assumed by the Sender account

    - Subscription Destination sends the logs to KDS (recipient stream)

7. Activity:

- Go to CloudWatch > Log Groups > select a Log Group > see Log streams

- Select a Log Stream, `/stdout` and `stderr`

- use search filter

- Go to Metrics Filter > Create metric filter

- Go to Subscription Filter > Create based on result

- CW > Logs Insights > run some query

8. Activity 2 (Live tail):

- Create Log group

- Craete log stream

- click *Start Tailing*

- Actions > Create log event

### CW Logs: Metric Filter

CW Logs can use filter expressions:

- eg. find a specifc IP inside of a log

- or count the occurences of "ERROR" in your logs

- metric filters can be used to trigger alarms

(IMPORTANT) Filteres do not retroactively filter data. Filter only publish the metric data points for event that happpen AFTER  the filter was created

- Ability to specify up to 3 Dimensions for Metric Filter (optional)

[CW Logs Agent] ---stream ---> [CW Logs] ---> Metric Filters --- integrate with ---> CW Alarms
(eg. if no. of "error" is more than x) ---> trigger SNS

1. Activity:

- CW lOgs > Log Group > Log Streams > Create Metric Filter

- Metric Filters > Select it > Create Alarm > choose send SNS topic

### All Types of Logs

1. Applications Logs

    - Logs produced by your app code

    - contains custom log messages, stack traces, so on

    - written to a local file on the filesystem

    - usually streamed to CloudWatch logs using a CW Agent on Ec2

    - if using *Lmabda*: direct integration with CloudWatch logs

    - if using *ECS or Fargate*, direct integration with CLoudWatch Logs

    - I fusing *Elastic Beanstalk*, direct integration with CLoudWAtch logs

2. OS Logs (event logs, system logs)

    - logs that are generated by your OS

    - informing you of system behaviour (eg. /var/log/messages or /var/log/auth.log)

    - usually streamed to CloudAwtch logs using CW Agent

3. Access Logs

    - List all the request for individual files that people have requesteed from a website

    - example for httpd: /var/log/apache/access.log

    - Usually for load balancers, proxies, web servers, etcs

    - AWS provides some access logs

4. AWS Managed Logs

    - Load Balancer Access Logs (ALB, NLB, CLB) => S3

        - access logs for your Load Balancers

    - CloudTrail Logs  => to S3 or CloudWatch Logs

        - Logs for API calls made within your account

    - VPC Flow Logs => to S3 and CloudWatch Logs

        - information about IP traiffic going to and from network interfaced in your VPC

    - Route 53 Access Logs => CW Logs

        - Log information about the queries that Route 53 receives

    - S3 Acces Logs => S3

        - server access logging provided detailed records for the requests that are made to a bucket

    - CloudFront Access logs => S3

        - Detailed information aboute every user request that Cloudfront receives

### CloudWatch Agent and CloudWatch Logs Agent

1. CloudWatch Logs for EC2

- by default, no logs from your EC2 machine will go to CW

- You need to run a CW Agent on EC2 to push the log files you want

- Make sure IAM permissions are correct

- CLoudWatch Log Agent can be setup on-premise too

2. CW Logs Agent (older want) and Unified Agent (newer one)

- both for virtual servers (EC2 instances, on-premise servers...)

- CloudWatch Logs Agent

    - Old version of the agent

    - can only send to CW Logs

- CloudAwtch Unified Agent

    - Collect additional system-level metrics suchas RAM, processes, etc..

    - Collect logs to send to CW Logs

    - Centralized configuration using SSM Parameter Store

3. CW Unified Agent - Metrics

    - Collected directly on your Linux server / EC2 instance

    - CPU (active, guest, idle, system, user, steal)

    - Disk Metrics (free, used, total), DiskIO (writes, reads, bytes, ios)

    - RAM

    - Netstat

    - Processes

    - Swap Space

    NOTE: Out-of-the-box metrics for EC2 - disk, CPU, network (high level)

### CloudWatch Alarms

Alarms are used to trigger notification for any metric

Various otions (sampling, % max, min, etc...)

Alarm states:

    - OK

    - INSUFFICIENT_DATA

    - ALARM

Period:

    - Length of tim ein seconds to evaluate the metric

    - High resolution custom metrics: 10 sec, 30 sec or multiples of 60 sec

1. Alarm Targets:

    a. Stop, Terminate, Reboot, or Recover an EC2 Instance

    b. Trigger EC2 Auto Scaling Action

    c. send Notification to SNS (from which you can do pretty much anything, when hooked with Lambda)

2. Composite Alarms

- CW Alarms are on a single metric

- Composite Alarms are monitoring the states of multiple other alarms

- AND and OR conditions

- Helpful ot reduce "alarm noise" by creating complex composite alarms


eg. [EC 2 instance] <---- monitor CPU ---- [CW Alarm A]


                    <---- minotor OPS ---- [CW Alarm B]

Cw Alarm A and Cw Alarm B is within a Composite Alarm, so if the alarms triggered OR or AND, then it triggers Amazon SNS

3. EC2 Instance Recovery

- Status Check

    - Instance status = check the EC2 VM

    - System Status = check the underlying hardware

    [EC2 instace] <--- minitor --- CW Alarm `StatusCheckFailed_System`

        - if triggered, CW Alarm can trigger *EC2 Instance Recoivery*

    - Recover: Same Private, Public, Elastic IP, metadata, placement group

4. Extras

- Alarms can be created based on CW Logs Metrics Filters

    [CW Logs with Metric Filter (if too many "ERROR" word)] -- trigger alarm --> [CW Alarm] --- alert ---> [SNS]

- To test alarms and notifications, set the alarm state to `Alarm` using CLI


```
aws cloudwatch set-alarm-state --alarm-name "my-alarm" --state-value ALARM --state-reason "testing purposes"
```

5. Activity

- Go to EC2 > Create EC2 Instance

- Go to CloudWatch > All alarms > Create alarm

- Under Metrics > Search for your EC2 instance > select it

- Sselect metric > CPUUtilization

- EC2 Action > if *In Alarm* > termiante this instance

- Open Cloud Shell > use `cloudwatch set-alarm-state` api to test alarm

### CloudWatch Synthetics Canary

Configurable script that monitor your API's, URLs, websites...

Reproduce what your customers do programmatically to find issues before customers are impacted

Check the availability and latency of your endpoints and can store load time data and screenshots of the UI

[Users] ---> [Route 53] ---> [EC2 (us-east-1)] <--- monitor --- [CW Synthetics Canary] --- if failed --- trigger ---> [CW Alarm] --- invoke ---> [Lambda Function] ---[update DNS record] ---> [Route 53] --- now points to ---> [EC2(us-west-2)]

Integration with CloudWatch Alarms

Script written in Nodejs or Python

Programmatic access to headless Google Chrome browser

Can run once or ona  regular schedule

1. CW Canary Blueprints

    a. Hearbeat Monitor - load URL, store screenshot and an HTTP archive file

    b. API Canary - test basic read and write functions of REST API

    c. Borken Link Checker - checks all links inside the URL that you are testing

    d. Visual Monitoring - comparea screenshot taken during a canary run with a baseline screenshot

    e. Canary Recorder - used with CLoudWatch Systhetics Recorder (record your actions on a website and automatically generates a script for that)

    f. GUI Workflow Builder - verifies that action s can be taken on your webpage (eg. test a webpage with a login form)



## 4.4 Amazon Athena

Serverless query service to analyze sata stored in Amazon S3

Uses standard SQL language to query the files (built on Presto)

Supports CSV. JSON, ORC, Avro, Parquet

Pricing: $5 per TB of data scanned.

Commonly used with Amazon Quicksight for reporting and Dashbooards

use cases: Business Intelligence / analytics / reporting, analyze & query VPC Flow Logs, ELB Logs, CloudTrail trails, etc... 

[load data] ---> [S3 bucket] <-- Query and Analyze -- [Amazon Athena] --- reporting and dashboar --> [QuichSight]

EXAM: analyze data in S3 using serverless SQL, use Athena

1. Athena Performance Improvement

- Use columnar data for cost-savings (less scan)

    - Apache Parquet ORC is recommended

    - Huge performance improvement

    - use Glue to convert your data to Parquet or ORC

- COmpress data for smaller retrievals (bzip2, gzip, lz4, snappy, zlip, zstd..)

- Partition datasets in S3 for easy querying on virtual columns

    - s3://yourBucket/pathToTable/<PARTITION_COLUMN_NAME>=<VALUE>

    - eg. s3://athena-examples/flight/parquet/year=1991/month=1/day=1

- Use larger files (> 128MB) to minimize overhead vs many smaller files

2. Federated Query

- Allows you to run SQL queries across data stored in relational, non-relational, object, custom data sources

- Uses *Data Source Connectors* that run on AWS Lambda to run Federated Queries (eg. CloudWatch Logs, DynamoDb, RDS)

[Amazon Athena] --> lambda (Data Source Conncted) ---> can connect to ElastiCache, DFocumentDB, DynamoDB, Redshit, Aurora, SQL Server, MySQL, etc...

3. Activity

- Go to S3 > create a bucket > get bucket name

- Go to Athena > Query Editor > Manage settings > insert your bucket in `location of query result` > Save

- [Refer steps here](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-s3-access-logs-to-identify-requests.html)

- Create database

- create external table

- RUn some queries