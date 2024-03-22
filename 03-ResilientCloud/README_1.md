# Domain 3: Resilient Cloud Solutions (Part 1)

## 3.1: lambda

### Lambda Versions

When we work on Lambda function, we work on the `$LATEST` version.

- `$LATEST` is mutable - meaning that you can edit and change the code.

When we are ready to publish, we create a version (v1, v2, etc).

- Versions are _immutable_ ie. we cannot update the code.

- Each Version is independent, will get their own ARN (Amazon Resource Name)

- VERSION = CODE + CONFIGURATION (nohting can be changed - immutable)

### Lambda Aliases

To give users a stable endpoint, use _AWS Lambda Aliases_

Lambda Aliases are pointers to Lambda function versions.

Can define "dev", "test", "prod" aliases and have them point at different lambda versions.

Aliases are mutable.

Users will interact with Aliases.

Aliases enable Canary deployment by _assigning weights_ to Lambda functions.

Aliases have their own ARNs.

Aliases cannot reference Aliases, can only reference Versions.

Activity:

1. Go to Lambda > Functions > Create Function

2. Go to Code > write some code > Click Deploy

3. Go to Action > Publish new version

4. Go to Aliases > Create Alias > Bind alias to a specific Version

5. Select an alias > Edit > You can assign weight (Canary deployment)

### Lambda Environment Variables

- key / value pair in "String" form.

- adjust the function behaviour without updating code

- env variables are available to your code

- Lambda Service adds its own system env vars as well.

- Helpful to store secrets - use KMS to encrypt

- Secrets can be encrypted by the Lambda service key or your own CMK

Lambda Functions > Configuration > Environment Variables

### Lambda Concurrency and Throttiling

Concurrency limit for Lambda: 1000 concurrent executions.

We can limit the max number of concurrency a Lambda function can have.

- Set a _reserved concurrency_ at the function level (=limit)

- Each invocation over the concurrency limit will trigger a *Throttle*

    - Throttle Behaviours:

        - If synchronus invocation => return `ThrottleError -429`

        - if asynchronus invocation => retry automatically and then go to DLQ

- If need higher limit, open a support ticket

1. Concurrency Issues:

- If dont reserve (=limit) concurrency, users accessing your apps via different methods (such as via API Gateaway and/or via SDK/CLI) will be throttled.

    - Concurrency limit applies to ALL lambda functions in your account. Therefore, we need to be careful when setting concurrency levels - higher is not always better

    - if one function goes over the limit, it might cause the other functions to be throttled

2. Concurrency and Asynchronus Invocations

    - if a function does not have enough concurrency available to process all events, additional requests are throttled.

    - for *throttling errors* (429) and *system errors (500 series)*, Lambda returns the event to the queue and attempts to run the function again for up to 6 hours.

    - The retry interval increases exponentially from 1 second after first attempt to a maxiumum of 5 minutes.

3. Cold Starts and Provisioned Concurrency

    - *Cold Start*

        - New instance => code is loaded and code outside the handler run (init)

        - if init is large (code, dependencies, SDK...) this process can take some time

        - First request served by new instances has higher latency than the rest

        - Can use Provisioned Concurrency

    - *Provisioned Consurrency*

        - Concurrency is allocated before the function is invoked (in advance)

        - so the cold start never happens and all invocations have low latency

        - Application Auto Scaling can manage concurrency (schedule or target utilization)

    - Notes:

        - Cold starts in VPC have been dramatically reduced in Oct & Nov 2019

    - Reserved Concurrency vs Provisioned Concurrency - see the diagram in lecture slides

4. Activity

- Go to Lambda > Funtions > Select a function > Go to Configuration Tab

- Select *Concurrency* from left NavBar > Edit

- Set Reserved Concurrency (eg. 20)

    - If a lambda function has a Reserved Concurrency of 20, then your account will have a remainder of 1000 - 20 = 980 Unreserved Concurrency

    - set to ZERO to enable always throttled.

- In Concurrency > Configuration : Add Provisioned Concurrency Configuration to reduce cold starts.

### Lambda File Systems Mounting

Lambda functions can access EFS file systems if they are running in a VPC.

Configure Lambda to mount EFS file systems to local directory during initialization.

Must leverage EFS Access Points.

Limitations: Watch out for the EFS connection limits (one function instance = one connection) and *connection burst limits*

Storage Options:

1. Ephermal Storage */tmp*

    - when Lambda function is destroyed, the storage gets destroyed

    - fastest acces speed

    - not shared across invocations

2. Lambda Layers

    - Durable ie. when Lambda Function is destroyed, the storage remains

    - fastest access aspeed

    - shared across invocations

3. Amazon S3

    - Flexible size

    - Durable

    - Fast access speed

    - supports Atomic with Versioning

    - Shared across invocations

4. Amazon EFS

    - Flexible Size

    - Durable

    - Very fast access spedd

    - shared across invocations

### Lambda Cross Account EFS Mounting

1. Account A >

    VPC A >

        Lambda Function

2. Account B >

    VPC B >

        EFS File System + Access Point

For lambda function in VPC A to mount to EFS File System in VPC B:

1. VPC A and VPC B must do *VPC Peering*

2. Make sure Lambda function has enough permissions to:

    - describe file systems

    - describe Mount targets

    - create access point

    - describe subnet

    - describe network interface

3. Apply an EFS File System Policy which allows the client from another account to mount on it.

```json
{
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "elasticfilesystem:ClientMount",
                "elasticfilesystem:ClientWrite"
            ],
            "Principal": {
                "AWS": "arn:the account nmber:root"
            }
        }
    ]
}
```

## 3.2 API Gateway

A common architecture:

[client] <----> [Load Balancer] <----> Lambda <---CRUD---> DynamoDB

Another option:

[client] <--REST API--> [API Gateway] <---Proxies Request to--> Lambda <---CRUD---> DynamoDB

- AWS ALmbda + API GAtewat = Full serverless, No Infrastructure to manage

- Support for the WebSocket Protocol

- Handles API versioning

- Handles different environments (dev, test prod)

- Handle security (Authentication and Authorization)

- Create API Keys, handles request throttling

- Import APIs to quickly define APIs using: Swagger/ OpenAPI definition file

- transform and validate requests and responses

- Generate SDK and API specifications

- Cache API responses

1. High Level Integration: AWS API Gateway and:

    - Lambda Function

        - Invoke Lambda Function

        - Easy way to expose REST API backed by AWS Lambda

    - HTTP

        - Expose HTTP endpoints in the backend

        - eg. internal HTTP API on premise, ALB...

        - why? Add *rate limiting*, caching, use authentications, API Keys, etc..

    - AWS Service

        - Expose any AWS API thorugh API Gateway

        - eg. start an AWS Step Function workflow, post a message to SQS

        - why? Add Authentication, deploy publicly, rate control...

    - Example: AWS API Gateway + AWs Service (Kinesis Data Streams): allow users to send data to Kinesis Data steams in a secure way without giving them access to AWS Credentials

    [client] ---request---> [API Gateway] --send--> Kinesis Data Streams ---send records to--> Kiensis Data Firehose ---store json files in---> S3

2. API Gatewat Endpoint Types:

- Edge Optimized (default): For global clients

    - requests are routed through the CloudFront Edge locations (improves latency)

    - API GAteway still lives in only one region

- Regional

    - For clients within the same region

    - Cloud manually combine with CloudFront (more control over the caching strategies and distrivbution)

- Private

    - can only be accessed from your VPC using an interface VPC endpoint (ENI)

    - to define access, use a *resource policy*

3. Security

- User authentication through:

    - IAM Roles (useful for internal applications)

    - Amazon Cognito (identity for external users - eg. mobile users)

    - Custom Authorizer (your own logic)

- Custom Domain Name HTTPS security through integration with *AWS Certificate Manager (ACM)*

    - if using Edge-Optimized endpoint, the certificate must be in *us-east-1*

    - if using Regional endppoint, the certificate must be in the API Gateway Region

    - Must setup CNAME or A-alias record in Route 53

4. Activity

- In Lambda > Create new Function > Copy the function's ARN

- Go to API Gateway > Build REST API (Public)

- In API > Create Method > Integration Type (Lambda function, http, *AWS Service*, VPC Link) > Insert the lambda Function's ARN

- Tick Lambda Proxy Integration

- Deploy API

- use Invoke URL

### API Gateway Stages and Deployment

Making changes to API Gateway does not make them effective immediately, until we Deploy.

Changes are deploye to "Stages" (as many as you want)

    - any name (eg. dev, test, prod)

    - each stage has own config parameters

    - each stage has own URL

    - stages can be rolled back as a history of deployments is kept

1. Stage Variables:

- like environment variables for API Gateway

- use them to change often chaning config values

- they can be used in:

    - Lambda function ARN

    - HTTP Endpoint

    - Parameter mapping templates

- use cases:

    - configure HTTP endpoints your stages talk to (dev, test, prod...)

        - use stage variable to indicate the corresponding Lambda Alias

    - pass config paramters to AWS Lambda through mapping templates

- Staging variables are passed to the "context" object in AWS Lambda

- Format: `[your function ARN]:${stageVariables.variableName}`

2. Activity:

- In API Gateway > Choose your API > Resources > Create Resource

### API Gateway: Open API Specification Integration

Common way to define REST APIs, using API defintion as code.

Import existing OpenAPI 3.0 Spec to API Gateway.

Can also export current API Gateway endpoints as OpenAPI spec.

OpenApi Specs are in Yaml or JSON.

Using OpenAPI we can generate client SDKs for our apps.

1. Use OpenApi spec to Validate Requests on your API Gateway

    - Configure API Gateway to perform basic validation of an API request before proceeding with the integration request.

    - When validation fails, API Gateway fails the request:

        - returns 400 error response

    - This reduces the unnecessary calls to the backend

    - eg check: required request params, query string, headers

2. How to do this

- Setup request validation by importing OpenAPI defintions file:

    - define validators

    ```json
    {
        "x-amazon-apigateway-request-validators": {
            "all": {
                "validateRequestBody": true,
                "validateRequestParameters": true
            },
            "params-only": {
                "validateRequestBody": false,
                "validateRequestParameters": true
            }
        }
    }
    ```

    - Enable params only on all API methods:

    ```json
    {
        "x-amazon-apigateway-request-validators": "params-only"
    }
    ```

    - Enable All validator on the POST /validation method - overrides the *params-only* validator inherited from the API.

    ```json
    {
        "paths": {
            "/validation": {
                "post": {
                    "x-amazon-apigateway-request-validators": "all"
                }
            }
        },
    }
    ```

3. Activity

- Go to API Gateway > Create REST API > Choose Import

- Go to API Gateway > Choose an API > Stages > Choose a Stage

- In *Stage Actions* > Export

- In *Stage Actions* > Generate SDK

### API Gateway: Caching API Responses

Caching reduce the numbner of calls made to the backend.

Default TTL (how long the result lives in the cache) - 300 seconds / 5 minutes

    - min is 0 (no caching)

    - max is 3600s / 1 hour

1 Cache per Stage

Possible to override cache settings per method

Cache can be encrypted.

Cache is expensive. Only makes sense on production.

1. Cache Invalidation

- Able to flush the entire cache (invalidate it) immediately - from the UI

- Or using Query Header: `header: Cache-Control:max-age=0`

    - requires IAM authorization:

    ```json
    {
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "execute-api:InvalidateCache"
                ],
                "Resource": [
                    "your arn..:api-id/stage-name/GET/path"
                ]
            }
        ]
    }
    ```

- Activity

    - API Gateway > Choose an API > Stages > Choose a Stage > edit

    - Under Cache Settings > enable `API Cache`

    - for a API specific path > edit > override Cache Settings configured in Stage Level

### API Gateway: Canary Depoyments

POssible to enable canary deployments for any stage (usually prod)

Choose the % of traffic the caanry channel receives

Metrics and Logs separate (for better monitoring)

POssible to override stage variables for canary

Equivalent to *blue/green* deployment with AWS Lambda & API Gateway

Eg:

Client --- 95% ---> API Gateway (Prod Stage)---> Lambda v1

Client ---  5% ---> API Gateway (Prod Stage Canary) ---> Lambda v2

1. Activity

- API Gateway > Choose an API > Resources > Craete Resource

- Create Method > lambda function

- Deploy API > New Stage > "Canary"

- In "Canary" tab > Create Canary > allocate request % distribution

- Deploy the API

- Promote Canary

### API Gateway: Logging and Tracing

1. CloudWatch Logs

    - Log contains information about request/response body

    - Enable CLoudWatch logging at the Stage Level (with Log level - ERROR, DEBUG, INFO)

    - Can override settings ona per API basis

    - Be careful - you will get alot of sensitive information into CloudWatch Logs

2. X Ray

    - Enable tracing to get extra information about requests in API GAteway

    - X Ray API Gateway + AWS Lambda gives you the full picture

3. CloudWatch Metrics

    - Metrics are by stage, Possibility to enable detailed metrics

    - types of metrics (IMPORTANT EXAM):

        - CacheHitCount & CacheMissCount: efficiency of the cache

        - Count: the total number API requests ina given period

    - IntegrationLatency: The time between when API Gateway relays a request tot he backend and when it receives a respinse from the backend

    - Latency: The time between when API Gateway receives a request from a client and when it returns a response to the client. The latency includes integration latency and other API Gateway overhead.

    - 4XXError (client-side) & 5XXError (server-side)

4. API Gateway Throttling

- Account limit

    - by default: API Gateway throttles requests at 1000rps across ALL API

    - Soft limit can be increased upon request

- In case of throttling => *429Too Many Requests* (retriable error)

- Can set *Stage limit* & *Method limits* to improve performace.

- Or you can define Usage Plans to throttle per customer

---> Just like Lambda Concurrency, one API tha tis overloaded, i fnot limited, can cause the other APIs to be throttled.

5. API GAtewat Errors

- 4xx means Client Errors

    - 400: Bad Request

    - 403: Access Denied, WAF filtered

    - 429: Quota Exceeded, Throttle

- 5xx means Server Errors

    - 502: Bad Gateway Exception, usually for an incompatible output returned from a Lambda proxy integration backend and occasionally for out-of-order invocations due to heavy loads

    - 403: Sevice Unavailable Exception

    - 504: Integration Failure- ex Endpoint Request Timed-out Exception

        - API GAteway requests timeout after 29 seconds maximum


### 3.3 AWS Elastic Container Service (ECS)

### ECS EC2 Launch Type

When we launch Docker containers on AWS = Launch *ECS Tasks* on *ECS Clusters*

EC2 Launch Type =  you must provision & maintain the infrastructure (the EC2 instances)

- Each EC2 instacen must run the *ECS Agent* to register in the ECS Cluster

- AWS takes care of starting / stopping containers

### ECS Fargate Launch Type

Launch Docker containers on AWS

- You DO NOT provision the infrastructure (no EC2 instances to manage)

- all serverless

- Just create *Task Definitions*

- AWS runs ECS Tasks for you based on the CPU/RAM you need

- to scale: just increase the number of Tasks - no more EC2 instances

### ECS IAM Roles

1. For EC2 Instance running an *ECS Agent* (EC2 LaunchType):

- EC2 Instance Profile

    - Used by ECS Agent

    - Makes API calls to ECS Service

    - Send container logs to CloudWatch logs

    - Pull Docker Image from ECR

    - Referene sensitive Data in Secrets Manager or SSM Parameter Store

2. For ECS Tasks (applicable to both EC2 LaunchType and Fargate):

- ECS Task Role:

    - allows each task to have specific role

    - each role allows you to be linked to different ECS services

    - Task Role is defined in the *Task Defintion*

### ECS Load Balancer Integrations

Client <---> Application Load Balancer (ALB) <---points to many---> ECS Tasks

- Application Load Balancer supported and works for most use cases. Works with Fargate.

- Network Load Balancer recommended only for high throughput / high performance use cases, or to pair it with AWS Private Link

- Classic Load Balancer: supported but not recommended (no advanced features, no Fargate)

### ECS Data Persistence: Data Volumes (EFS)

We may want to mount EFS File systems onto ECS Tasks

Works for both EC2 and Fargate launch Types.

Fargate + EFS = Ultimate serverless

Use cases: persistent multi-AZ shared storage for your containers

Note: Amazon S3 CANNOT be mounted as a File System

### ECS: Service Auto Scaling

Automatically increase/decrease the desired number of ECS Tasks.

Use AWS Application Auto Scaling on:

    - ECS Service *Average CPU Utilization*

    - ECS Service *Average Memory Utilization* - Scale on RAM

    - ALB *Request Count Per Target* - metric coming from the ALB

*Target Tracking* - scale based on target value for a specific CloudWatch metric

*Step Scaling* - scale based on a specified CloudWatch Alarm

*Scheduled Scaling* - scale basen on specified date/time (predictable changes)

ECS Service AutoScaling (task-level) is NOT EQUAL to EC2 Auto Scaling  (EC2 Instance Level)

Fargate Auto Scaling is much easier to setup, since it is serverless.

1. Auto Scaling EC2 Launch Types

- Accommodate ECS Service Scaling by adding underlying EC2 Instances

- *Auto Scaling Group Scaling*

    - Scale your ASG based on CPU utilization

    - Add EC2 instance over time

- *ECS Cluster Capacity Provider* (better way)

    - Used to automatically provision and scale the infra for your ECS Tasks

    - Capacity Provider paired with an ASG

    - Add EC2 Instances when you're missing capacity (CPU, RAM...)

Example:

1. An ASG has Service A, and Service A has Task 1 and Task 2.

2. When CPU Usage increases, this will inform CloudWatch Metric (ECS Service CPU Usage)

3. CloudWAtch Metric triggeres CloudWatch Alarm, and this will trigger Scale activity

4. Capacity increase --> New Task will be added.

5. (optional) if the service is running on EC2 Launch Type, *Scale ECS Capcity Providers* will help

### ECS: Solution Architectures

1. ECS Tasks Invoked by EventBridge

- Client/users upload objects to S3 bucket

- S3 bucket emits event to EventBridge

- EventBridge configured with Rule to Run ECS Task

- ECS Task Role (Access S3 and Dynamo DB)

- ECS Task can get the Object from S3, process it, then save result in Dynamo DB

2. ECS Tasks invoked by Event Bridge Schedule

- Configure EventBridge to schedule, every 1 hour, to trigger process

- Rule: Create New / Run ECS Task

    - eg. to create ECS Task Role, Access S3

- Our app can do some Batch Processing against files in S3

3. ECS with SQS Queue

- Service A on ECS, with ECS Task 1 and ECS Task 2

- Messages are sent to SQS Queue

- Service A can Poll for messages from SQS Queue, and process them

- ECS Service Auto Scaling can add number of Tasks within Service A to increase capacity

4. ECS to INtercept Stopped Tasks using Event Bridge

- Let's say if any of your Tasks exitting, it emits event to EventBridge

- EventBridge can trigger SNS

- SNS can send email to Administrator

### ECS Logging with awslogs driver

Containers can send logs directly to CloudWatch Logs

You need to turn on `awslogs` log driver (for CW Logs)

Configure logConfiguration parameters in your Task Definition

```json
{
    "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
            "awslogs-group": "/ecs/fargate-task-defintion",
            "awslogs-region": "us-east-1",
            "awslogs-stream-prefix": "ecs"
        }
    }
}
```

1. If use Fargate LaunchType:

    - Task Execution Role must have the required permissions

    - Supports awslogs, splunk, awsfirelens log drivers

2. If use EC2 LaunchType

    - Prevents logs from taking up disk space on your container EC2 instances

    - Uses CloudWatch Unified Agent & ECS Container Agent

    - Enable logging using `ECS_AVAILABLE_LOGGING_DRIVERS` in `/etc/ecs/ecs/.config`

    - Container EC2 instances must have permissions

3. Logging with Sidecar Container

    - Using a sidecar container which is responsible for collecting logs from all other containers and files on the file system and send the logs to a log storage

    - for example, send the logs to CloudWatch Logs

### Amazon ECR: Elastic Container Registry

Store and manage Docker images on AWS

Private and Public repository (Amazon ECR Public Gallery)

Fully integrated with ECS, backed by Amazon S3

To pull Docker images from an ECR Repo, than instance must have access through IAM/Policies.

Supports image vulnerability scanning, versioning, tags, lifecycle..

1. Lifecycle Policies

- automatically remove old or unused images based on age or count

    - eg. expire untagged images older than 14 days

    - eg. keep only one untagged image and expire all others

- Each Lifecycle policy contains one or more rules

- All rules are evaluated at the same time, then applied based on priority

- Images are expired within 24 hours after they meet the criteria

- reduce unnecessary storage costs

2. Uniform Pipeline

- Code Pipeline: Code Commit (in any language) --- push code ---> CodeBuild (build Docker Images)

- Docker Images are pushed to ECR

- ECS or Fargate pulls those images


## 3.4 Amazon Kubernetes Service (EKS)

Launch and manage Kubernetes clusters on AWS.

It is an alternative to ECS. Similar goal but different API.

EKS Launch MOdes:

- EC2: for worker nodes

- Fargate: to deploy serverless containers

Use Case to use:

- if your companyu is already using Kubernetes on premise or in another cloud, wants to migrate to AWS using Kubernetes

- Kubernetes is Cloud Agnostic - can be used with any cloud service: Azure, GCP

1. Basic Architectures:

Each Availability Zone:

- One Public Subnet

    - EKS Public Load Balancer (ELB) for client access

- One Private Subnet

    - 1 EKS Node

        - Many EKS Pods

    - Nodes across Availability Zone is managed by Auto Scaling Group (ASG)

    - Can have Private ELB / Load Balancer to inter service communication

2. Node Types

- Managed Node Groups

    - Creates and manages Nodes (EC2 instances) for you

    - Nodes are part of an ASG managed by EKS

    - Supports On-Demand or Spot Instances


- Self-Managed Nodes

    - Nodes created by you and registered tot he EKS cluster and managed by an ASG

    - You can use prebuilt AMI - Amazon EKS Optimized AMI

    - Supports On-Demand or Spot Instances

- AWS Fargate

    - No maintenance required; no nodes managed

3. Data Volumes

- Need to specify `StorageClass` manifest on your EKS cluster

- Leverages a Container Storage Interface (CSI) compliant driver

- Support for:

    - Amazon EBS

    - Amazon EFS (works with Fargate)

    - Amazon FSx for Lustre

    - Amazon FSx for NetApp ONTAP

### EKS Logging

1. Control Plane Logging

- Send EKS Control Plane audit and diagnosttic logs to CloudWatch Logs

- EKS COntrol Plane LogTypes

    - API Server (api)

    - Audit (audit)

    - Authenticator (authenticator)

    - Controller Manager (controllerManager)

    - Scheduler (scheduler)

- ability to select the exact log types to send to CloudWatch Logs

2. Nodes and Containers Logging

- You can capture node, pod, containers logs and send them to CloudWatch Logs

- Use *CloudWatch Agent* to send METRICS to CloudWatch

- Use *FluentBit* or *Fluentd* log drivers to send LOGS to CloudWatch logs

- Contianer logs are stored on a Node directory `/var/log/containers`

- Use *CloudWatch Container Insights* to get a dashboarding monitoring solution for nodes, pods, tasks, and services

## 3.5 Amazon Kinesis

Makes it east to collect, process and analyze streaming data in real time

Ingest real-time data such as: Application logs, Metrics, Website clickstreams, IoT telemetry data

4 Services:

1. (most important) Kinesis Data Streams: capture, process and store data streams

2. Kinesis Data Fireshose: Load data streams into AWS data stores

3. Kinesis Data Analytics: analyze data streams with SQL or Apache Flink

4. Kinesis Video Streams: capture, process and store video streams

### Kinesis Data Streams

Stream Big Data in your system.

Stream has Shards. Shards are numbered: Shard 1, Shard 2... Shard N. 

You must provision Shards AHEAD OF TIME.

Properties:

1. Retention between 1 day to 365 days

    - Ability to reprocess (replay) data

2. Once data is inserted to Kinesis, it cannot be deleted (immutability)

3. Data that shared the same partition (ie Partition Key) goes to the same Shard (ordering)

Components:

1. PRODUCERS: Send data into Kinesis Data Stream

    - Could be Appls, Client, SDK, KPL (Kinesis Producer Library), Kinesis Agent

    - Produce Records

2. RECORDS: entity sent by PRODUCER to KDS

    - Record has Partition Key + Data Blob (up to 1MB)

    - Records are sent at 1MB/sec or 1000 msg/sec per shard

3. CONSUMERS: consume Records in KDS

    - Can be in many forms: Apps, KCL (Kinesis Client Library), SDK, Lambda, Kinesis Data Firehose, Kinesis Data Analytics

    - Records consumed has:

        - Partition Key

        - Sequence no.

        - Data Blob

    - Throughput: 
    
        - 2MB/sec (shared) Per Shard all consumers OR

        - 2MV/sec (enhanced) Per Shard per consumer


[PRODUCER] --- [RECORD] ---> [KDS] --- [RECORD] ---> [CONSUMER]

Capacity Modes:

1. Provisioned mode: suitable if you can plan capacity in advance

- Choose the number of shard provisioned, scale manually or using API

- each Shard gets 1MB/s in (or 1000 records per second)

- Each Shard gets 2MB/s out (classic or enhanced fan-out consumer)

- Pay per Shard provisioned per hour

2. On Demand Mode: suitable if you dont know capacity in advance

- No need to provision or manage the capacity

- Default capacity provisioned (4MB/s in or 4000 records per second)

- Scales automatically based on observed throughput peak during the last 30 days

- Pay per stream per hour & data in/out per GB

Security:

- control access/authorization using IAM policies

- Encryption in flight using HTTPS endpoints

- Encryption at rest using KMS

- You can implement encryption/decryption of data on client side (harder)

- VPC Endpoints available for Kinesis to access within VPC

- MOnitor API Calls with CloudTrail

Activity:

- In Kinesis > Create Kinesis Data Stream

- Select capacity mode

- Use SDK for producer and SDK for consuming

- Open CloudShell


```bash
#!/bin/bash

# get the AWS CLI version
aws --version

# PRODUCER

# CLI v2
aws kinesis put-record --stream-name test --partition-key user1 --data "user signup" --cli-binary-format raw-in-base64-out

# CLI v1
aws kinesis put-record --stream-name test --partition-key user1 --data "user signup"


# CONSUMER 

# describe the stream
aws kinesis describe-stream --stream-name test

# Consume some data
aws kinesis get-shard-iterator --stream-name test --shard-id shardId-000000000000 --shard-iterator-type TRIM_HORIZON

aws kinesis get-records --shard-iterator <>
```

Scaling Consumers

- `GetRecords.IteratorAgeMilliseconds` (CloudWatch metric)

    - difference between current time and when the last record of trhe GetRecords call was writtern tot he steam

    - Used to track the progress of Kiesis Consumers (track the read position)

    - IteratorAgeMilliseconds=0, then records being read are completely caught up with the Stream

    - IteratorAfeMilliseconds > 0 means we're not processing the records fasts enough (lag)

    [KDS] --- IteratorAgeMiliseconds metric ---> CloudWatch Metrics ---> trigger CloudWatch Alarm if IteratorAgeMiliseconds = 5 minutes ---> CloudWatch Alarm triggeres scale out to our ASG.


### Kinesis Data Firehose

Takes data from Producers (Apps, Client, SDK, Kinesis Agent, Kinesis Data Stream, Amazon CloudWatch (Log and Events), AWS IoT), optionally transform the data using Lambda, then write in Batches to Destinations. You can then send ALL or only the failed data to an S3 bucket.

Types of destinations:

1. AWS Destinations:

- AWS S3

- Amazon Redshift (copy through S3)

- Amazon OpenSearch

2. Third Party Partner:

- Datadog

- Splunk

- MongoDb

3. Custom Destinations

- HTTP Endpoint

Fully managed service, no admin, automatic scaling, serverless

Pay for data going through Firehose

Near Realtime:

- Buffer interval: 0 seconds (no buffering) to 900 seconds

- Buffer size: minimum 1MB

Supports many data formats, conversions, transformations, compression

Supports custom data transformations using AWS lambda.

Activity:

- Go to Amazon Kinesis > Data Firehose

- Can convert Record Formats

- IAM Role permissions to write to the target (S3 bucket)

### Exam: Kinesis Data Stream vs Firehose

1. Data Stream:

- streaming service for ingest data at scale

- write custom code (producer / consumer)

- real time

- Manage scaling (shard splitting / merging)

- Data storage for 1 to 365 days

- supports REPLAY capability

2. Firehose

- (Ingesting service) Load streaming data into S3/ Redshift/ OpenSearch/ Third Party / Custom HTTP

- Fully managed

- near realt time

- Automatic scaling

### Kinesis Data Analytics

1. For SQL Applications

- Real time analytics

- Sources: Kinesis Data Streams, or Kinesis Data Firehose.

- Use SQL Statements to perform advanced analytics

- can add reference data from S3 to enrich streaming data

- Can send to Destinations:

    - Kinesis Data Stream

        - real time processing with lambda or apps running on EC2

        - create streams out of real-timem analytics queries

    - Kinesis Firehose:

        - Send to S3, RedShift, etc..

        - Send analytics query results to destinations

- Fully managed, no servers to provision

- Pay for actual consumption rate

- Use cases:

    - Time-series analytics

    - Real-time dashboards

    - real-time metrics

2. Kinesis Data Analytics for Apache Flink (aka Amazon Managed Service for Apache Flink)

- Use Flink (Java, Scala, SQL) to process and analyze streaming data

- read from sources:

    - Kinesis Data Streams

    - Amazon MSK

- run any Apache Flink application on a managed cluster on AWS

    - automatic provisioning of computing resources, parallel computation, automatic scaling

    - application backups (implemented as checkpoints and snapshots)

    - Use any Apache Flink programming features

    - Flink does not read from Firehose (use Kinesis Analytics for SQL)

- Activity:

    - Go to Kinesis > Managed Apache Flink

### Kinesis Analytics Machine Learning

Perform ML on Kinesis Data Analytics

1. RANDOM_CUT_FOREST

- SQl function used for *anomaly detection* on numeric columns in a stream

- eg. Detect anomalous subway ridership during the NYC marathon

- uses recent history to compute model

2. HOTSPOTS

- locate and return information about relatively dense regions in your data

- eg. a collection of overheated servers ina data center