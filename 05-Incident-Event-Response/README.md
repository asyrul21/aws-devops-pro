# Domain 5: Incident and Event Response

## 5.1 EventBridge

Formerly known as CloudWatch Events.

We can:

- Schedule Cron jobs (scheduled scripts)

    - every hour --> trigger script on Lambda function

- Event Pattern: Event rules to react to a service doing something

    - IAM Root User Sign in event --> SNS Topic with email notification

- Trigger Lambda functions, send SQS/SNS messages

1. EventBridge Rules

[source] --(filter events (optional))--> [EventBridge] -- generates JSON doc--> [Destinations]

- Example Source:

    - EC2 Instance, CodeBuild, S3 Event (eg. upload an object), Trusted Advisor (eg. new Finding), CLoudTrail (any API call)

    -Scheduler or Cron (eg. every 4 hours)

- Destinations

    - Compute

        - Lambda, AWS Batch, ECS Task

    - Integration

        - SQS, SNS, Kinesis Data Streamss

    - Orchestration

        - Step Functions, CodePipeline, CodeBUild

    - maintenance

        - SSM, EC2 Actions

2. When AWS services as source, EventBridge is acting as *default Event Bus*. When integrated with AWS SaaS Partners like Zendesk, Datadog, EventBridge acts as *Partner Event Bus*. You can integrate with Custom Apps, then EventBridge acts as *Custom Event Bus*

    - Event buses can be accesses by other AWS accounts using Resource-based Policies

    - You can archive events (all/filter) sent to an event bus (indefinitely or set period)

    - ability to replay archived events

3. Schema Registry

- EventBridge can analyze the events in your bus and infer the schema

- The *Schema Registry* allows you to geenerate code for your app, that will know in advance how data is structured in the event bus

- schema can be versioned

3. Resource-Based Policy

- manage permisions for a specific EventBus

- eg. allow/deny events form another AWS account or AWS region

- use case: aggregate all events from your AWS Organization in a single AWS account or AWS region

```json
{
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "events:PutEvents",
            "Principal": { "AWS": "12312313"},
            "Resource": "ard:aws:...event-bus/central-event-bus"
        }
    ]
}
```

The Policy above will allow *events* from another AWS account

4. Activity

- Go to EventBridge > event buses > Create your own event bus > *central-event-bus-v2*

- Define Resource-based policy for cross account access

- setup source with Auth0

- Create Rule > select a bus

- Build event pattern > select source > other > sample event > EC2 State-Changed Notification

- Schemas > aws.ec2@EC2InstanceStateChangeNotification > Download code bindings in programming languages

5. Activity: Content Filtering

- EventBridge > Create Rule > AWS Events > type: S3 Object Created

- event Pattern > type: all > Edit Pattern

 - add the `detail-type: Object created` to the pattern json - copy from Sample Event

    - Prefix matching > insert

    - numeric mathcing, etc

6. Activity: Input Transformer

- EventBridge > Create Rule > type: Rule  with an event pattern

- source: AWS events

- sample events: *EC2 Instance State-change Notification*

- Creation Method > Use pattern form

- Event pattern > source: AWS Service, service: EC2, type: EC2 instance State-Change Notification

- Any state > Next

- Target > CloudWatch log group > `aws/events/event-input-transformation`

- Addition settings > Configure Target Input > Input Transformer > Configure Input Trasnformer

- set Sample Event > Input Paths > Target Input Transformer

```json
{
    "instance": "$.detail.instance-id",
    "resource": "$.resources[0]"
}
```

- template:

```json
{
    "ins": <instance>,
    "res": <resource>,
    "message": "<instance> using resource <resource>"
}
```

- confirm and save event

- EC2 > Launch an instance

- Check on CloudWatch Logs

## 5.2 S3 Event Notifications

Events:

- S3:ObjectCreated, S3:ObjectRemoved, S3:ObjectRestore, S3:Replication

- Object name filtering possible (*.jpg)

- Use case: generate thumbnails of images uploaded to S3

- can create as many S3 Events as desired

- S3 event notificaitions typically deliver events in seconds but can sometimes take a minute

---events---> [S3]

                ---> SNS

                ---> SQS

                ---> Lambda Function

                ---> event bridge

1. IAM Permissions

- eg. --events--->[S3] ---> [SNS]

- we need to attach SNS Resource (Access) Policy

- same applies if we point S3 to SQS/Lambda Function

2. S3 Event Notification with Event Bridge

---events---> S3 ---all events---> [EventBridge] (by default) --rules---> send over 18 AWS servivces as destination

- advanced filtering options with JSON rules (metadata, object size, name..)

- multiple Destinations: eg. Step Funtions, Kinesis Data Streams / Firehose

- EventBridge Capacilities - archive, Replay events, Reliable delivery

3. Activity

- SQS > Create Queue > Acces policy > Allow S3 to write to SQS Queue > Edit > Policy Generator

    - type: SQS Queue Policy

    - effect: allow

    - service: SQS

    - actions: send message

    - arn

- S3 > Create Bucket

- S3 > bucket > properties > Event Notification > EventBridge > Edit > On OR

- Create Event Notifiaction > type: All Object create events

- Destination > SQS Queue > use your SQS Queue

- S3 > upload file

## 5.3 S3 Object Integrity

S3 uses checksum to validate the integrity of uploaded objects

1. using MD5

    [user (calculated MD5 (y))] ---> user --- upload (content-MD5) ---> [S3 (S3 Calculated MD5: x)]

- if user calcualted MD5 == S3 calculated MD5 : upload OK

- otherwise: upload will fail

- can pass in Header named Content MD5

2. Using MD5 and Etag

- ETag - represents a specific version of the object, ETag = MD5 (if SSE-S3)

    [user] <---get object metadata (ETag: abc..)-- [S3 (Object(ETag: abc..))]

- if User calcualted MD5 = Object Etag: OK objects are same

- otherwise, no

3. Other supported checksums: SHA-1, SHA-456, CRC32, CRC32C

## 5.4 Health Dashboard


1. Health Dashboard - Service History

- Shows all regions, all services health

- Shows historical information for each day

- Has an RSS feed you ca subscribe to

- previous called AWS Service Health Dashboard

2. Health Dashboard - Your Account

- Previosuly known as AWS Personal Health Dashboard (PHD)

- AWS Account Health Dashboard provides alerts and remediation guidance when AWS is experiencing events that may impact you

- Service Health Dashboard - displays general status of AWS services. Account Health Dashboard gives *personalized* view into the *performance* and *availabuility* of the AWS services underlying your AWS resources

- The Dashboard displays relevant and timely information to help you manage event in progress and provides *proactive notification* to help you plan for scheduled activities

- can aggregate data from an entire AWS organization 

- Click on top right corner - global service

- alerts, remediation, proactive, scheduled activities

3. Activity

- Click on Bell on top right

- Service History

### Health Dashboard: events and Notifications

1. Event Notification

- Use EventBridge to react to changes for AWS Health events in your AWS account

- eg. receive email notification when EC2 instance in your AWS account are scheduled for updates

- This is possible for Account events (resources that are affected in your account) and Public Events (regional availability of a service)

- Use cases: send notifications, capture event information, take corrective action

2. examples

    - Remediating Exposed IAM Access keys

        [AWS Health Dashboard] ---(event: Exposed Access Keys) --> [EventBridge] -- invoke --> [lambda] --delete access keys---> [IAM]

    - Restarting Instances that are scheduled for retirement

        [AWS health dashboard] --- (event:Instance scheduled for retirement) --> [EventBridge] -- restart instance ---> [EC2 ]

## 5.5 EC2 Instance Status Checks

Automated checks to identify hardware and software issues

2 types:

1. system status checks

    - monitors problems with AWS systems (software/hardware issues on the physical host, loss of system power...)

    - Check *Personal Health Dashboard* for any scheduled critical maintenance by AWS to your instance's host

    - Resolution: stop and start the instance (instance migrated to a new host)

2. Instance status checks

    - monitors software/network configuration of your instance (invalid network config, exhauisted memory...)

    - Resolution: Reboot the instance or change instance ocnfiguration

Automating Recovery

1. CW Metrics:

- 1 minute interval

- StatusCheckFailed_System

- StatusCheckFailed_Instance

- StatusCheckFailed (for both)

- Option 1: CloudWatch Alarm

    - Recover EC2 instance with the same private/public IP, EIP, metadata and Placement Group

    - Send notification using SNS

[EC2 Instacne] <-- monitor (StatusCheckFailed)-- [CloudWatch] --- trigger --> [CW Alarm]
                                                                                
[CW Alarm] 

    --> Recover EC2

    --> Send SNS notif


- OPtion 2: Auto Scaling Groups

    - Set min/max/desired of 1 to recover an instance but wont keep the same private and elastic IP


Activity:

1. EC > select instance > Status Checks tab

2. Action > Create status check Alarm

3. Alarm action > recover or reboot

4. Alarm theshold

5. Launch CW alarm and set state API using CloudShell

## 5.6 CloudTrail

Provides governance, compliance, and audit for your AWS Account

CouldTrail is enabled by default

Get a history of events / API calls made within your AWS Account by:

- Console

- SDK

- CLI

- AWS Services

Can put logs from CloudTrail info CoudWatch Logs or S3

A trail can be applied to ALl Regions (default) or a single Region

I f a resource is deleted in AWS, investigate CloudTrail first

[SDK/CLI/Console/IAM Users & Roles] ---> [CloudTrail Console (inspect and audit)] 
    
    ---> [CloudWatch Logs]

    OR

    ---> S3 Bucket

1. CloudTrail Events

- Management Events:

    - operations that are performed on resources in your AWS account

    - examples:

        - configuring security (IAM AttachRolePolicy)

        - configure rules for routing data (Amazon EC2 *CreateSubnet*)

        - Setting up logging (AWS *CloudTrail*)
    
    - By default, trails are configure to log management events

    - Can separate *Read Events* (that dont modify resources) from *Write Events* (that may modify resources)

- Data Events

    - By default, data events are not logged (because high volume operations)

    - Amazon S3 object-level activity (ex *GetObject*, DeleteObject, PutObject): can separate Read and Write Events

    - AWS Lambda function execution activity (the Invoke API)

- CloudTrail Insights Events

2. CloudTrail Insights

- Enable CloudTrail Insights to detect unusual activity in your account:

    - innacurate resource rpovisioning

    - hitting service limits

    - Bursts of AWS IAM actions

    - Gaps in periodic maintenance activity

- CloudTrail Insights analyzes normal management events to create a baseline

- And then continuously analyze *write* events to detect unusual patterns

    - anomalies appear in the CloudTrail console

    - Event is sent to Amazon S3

    - An EventBridge event is generated (for automation needs)


[Management Events] ----  continuously analyzed by ---> [CloudTrail Insights]

[CloudTrail Insights] --- generate ---> [Insights Events]

[Insights Events]

    ---> [CloudTrail Console]

    ---> S3 Bucket

    ---> EventBridge Event

3. CloudTrail Events Retention

- events are stored for 90 days in CloudTrail

- to keep events beyond this perios, log them to S3 and use Athena


[management events / data events / insights events] ---> [CloudTrail (90 days)] ---log---> [S3 Bucket (for long term retention)] ---analyze---> [Athena]

4. Activity

- Go to CloudTrail > event History

- EC2 > terminate an instance

- On CloudTrail > event History > see the event *TerminateInstances*

### CloudTrail and Event Bridge Integration

Examples:

1. [User] -- *DeleteTable* API call ---> [DynamoDB] -- all API calls logged to --> [CloudTrail]

[CloudTrail] -- generates events --> [Amazon EventBridge] -- alert (create Rule for *DeleteTable*) --> [SNS]

2. [User] -- AssumeRole --> [IAM] -- API Call logs--> [CloudTrail] --event--> [EventBridge]--> [SNS]

3. [User] -- Edit SecurityGroup Inbound Rules *AuthorizeSecurityGroupIngress* --> [EC2 Security Group]

[EC2 Security Group] -- API Call Logs --> [CloudTrail] -- event --> [EventBridge] --> [SNS]

## 5.7 SQS Dead Letter Queue (DLQ)

If a consumer fails to process a mesage within the VisibilityTimeout..

- The mesage goes back to the queue

We can set a threshold of how maby times a message can go back to the queue

After the *MaximumReceives* threshold exceeded, the message goes into a dead letter queue (DLQ)

[SQS Queue] ---> [Consumer] -- if fail, goes back to --> [SQS Queue (set threshold here)] --if exceeds threshold -- send to --> Dead Letter Queue


- Useful for debugging

1. IMPORTANT

- DLQ of a FIFI queue must also be a FIFO queue

- DLQ of a Standard Queue must also be a Standard Queue

- Make sure to process the messages in the DLQ before they expire

    - Good to set a retention of 14 days in the DLQ

2. Redrve to Source feature

- feature to help consume messages in the DLQ to understand what is wrong with them

- when our code is fixed, we can redrive the messages from the DLQ back into the source queue (or any other queue) in batches without writing custom code

Manual Inspectiona and debugging --> [DLQ] --redrive task--> [SQS Source Queue] --> [Consumer]

3. Activity

- SQS > Create Queue

- Choose another queue > edit > change VisibilityTimeout > Dead letter Queue > Choose the one you created > adjust *maximum receives*

- Poll for messages

- In DLQ > Start DLQ Redrive > Redrive to Source

## 5.8 SNS Dead Letter Queue (DLQ)

After exhausting the delivery policy (delivery retries), messages that havent been delivered are discarded unless you set a DLQ (Dead Letter Queue)

Redrive Policy - JSON object that refers to the ARN of the DLQ (SQS or SQS Fifo)

[SNS] --subscribed by --> [HttpEndpoint or Lambda]

if fail -- send to --> DLQ

- DLQ is attached to SNS Subscription-Level (rather than the SNS topic)

## 5.9 AWS X Ray

Visual analysis of our applications.

- Trace requests across your microservices (distributed systems)

- integrations with:

    - EC2 - install the X Ray Agent

    - ECS - install the X Ray agent or Docker container

    - Lambda

    - Beanstalk - agent is automatically installed

    - API Gateway - helpful to debug errors (such as 504)

- The X Ray agent or services need IAM permissions to X Ray

### X Ray with ElasticBeanstalk

EBS include XRay Daemon by default

You can run the daemon by setting an option in Elastic Beanstalk console or with a configuration file (in `.ebextensions/xray-daemon.config`)

```js
option_settings:
    aws:elasticbeanstalk:xray:
        XRayEnabled: true
```

IMPORTANT:

- Make sure to give your instacne profile the correct IAM permissions so that X Ray daemon can function correctly

- then make sure your application code in instrumented with the X Ray SDK

- The X Ray daemon is not provided for Multicontainer Docker

1. Activity

- EBS > Create Application > Nodejs > Configure more options

- Software > edit > Enable X Ray Daemon

- Security > Edit > virtual machine permissions > IAM Instance Profile > *aws-elasticbeanstalk-ec2-role* > Check for X Ray Role permissions

## 5.10 AWS Distro for OpenTelemetry

Secure, production-ready AWS-supported distribution of the open-source project Open Telemetry project

Provides a single set of APIs, libraries, agents, and collector services

Coolects distributed traces and metrics from your apps

Collect metadata from your AWS resources and services

*Auto-instrumentation Agents* to collect traces without changing your code

Send traces and metrics to multiple AWS services and partner solutions

    - X ray, CloudWatch, Prometheus

- Instrument your apps running on AWS (eg EC2, ECS, EKS, Fargate, lambda) and on premises

- IMPORTANT: Migrate from X Ray to AWS Distro for Telemtry if you want to standardize with open-source APIs from Telemtry or send traces to multiple destinations simultaneously

1. Collect Traces: data about the requests from each app the request passes through

2. Collect Metrics: collect metrics from each app the request passes through

3. AWS Resources and Contextual Data: collect info about AWS resources and metadata where the app is running

All of above can be sent to:

- X ray

- CloudWatch

- Amazon Managed Service for Prometheus














    















