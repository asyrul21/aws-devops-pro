# Domain 3: Resilient Cloud Solutions (Part 2)

## 3.6: Route 53

A highly available, scalable, fully managed and Authorotive DNS.

Authorotive: you can update the DNS records.

Also a Domain Registrar.

Ability to check the health of your resources.

The only AWS service which provides 100% availability SLA.

### Route 53 Records

How you want to route traffic for domain.

Each record contains:

- Domain/subdomain Name - eg. example.com

- RecordType - supports (EXAM) A / AAAA / CNAME / NS

- Value - eg. 12.34.56.78

- Routing Policy - how Route 53 responds to queries

- TTL - amount of time th record cached at DNS Resolvers

RecordTypes:

- A: maps a hostname to IPv4

- AAAA: maps hostname to IPv6

- CNAME: maps a hostname to another hostname

    - target is a domain name which MUST have an A or AAAA record

    - Cant create a CNAME record for the top node of a CNS namespace (Zone Apex)

        - eg. Cant create for `example.com` but can for `www.example.com`

- NS: Name servers for the Hosted Zone

    - Control how traffic is routed for a domain

Hosted Zones

- A container for records that define how to route traffic to a domain and its subdomains

- Public Hosted Zones 

    - contains records that specify how to route traffic on the internet (public domain names)

    - can answer queries from public clients

- Private Hosted Zone

    - contains records that specify how you route traffic within one or more VPCs (private domain name)

    - within VPC

    - allow identify private resources with private domain names

You pay $0.50 per month per hosted zone

### Route 53: Routing Policy - Weighted 

Control the % of the requests that go to each specific resource.

Weights dont need to sum to 100;

Assign each record a relative weight:

traffic (%) = weight of specific record / sum of all the weights for all records.

DNS records must have the SAME NAME and TYPE

can be associated with Health checks.

Use cases:

- Load balancing between regions

- testing new application versions

Assign weight of 0 to a record to stop sending traffic to the resource

If all records have weight of 0, then all records will be returned equally

Activity:

- Go to Route 53 > Hosted Zones > choose your hosted zone > create Record

- In Routing Policy, select Weighted

### Route 53: Routing Policy - Latency-based

Redirect to the resource that has the least latency (closest to us).

Helpful when latency for users is a priority

Latency is based on traffic between USERS and AWS Regions

Germany users may be directed to the US (if that's the lowest latency)

Can be assocaited with Health Checks (has failover capability)

Activity:

- Go to Route 53 > Hosted Zones > choose your hosted zone > create Record

- In Routing Policy, select Latency

- MUST define REGION

### Route 53: Routing Policy - Fail-over (Active-Passive)

Route 53 needs to do health checks to an instance. If it fails, Route 53 automatically swaps out the failed instance with the other instance.

Activity:

- Go to Route 53 > Hosted Zones > choose your hosted zone > create Record

- In Routing Policy, select Failover

- MUST associate a health check

- Failover type - one must be Primary, the other Secondary

## 3.7 RDS Read Replicas vs Multi Availability Zone (AZ)

RDS: Help you get managed Relational Databases.

Read replicas help us scale READS.

We can create Up to 15 Read Replicas.

Types:

- Within AZ

- Cross AZ

- Cross Region

Replication is ASYNC, betweent the main RDS instance (accepts read and writes) and read replicas.

Reads are EVENTUALLY CONSISTENT.

Replicas can be promoted to their own DB.

Apps must update connection string to leverage read replicas.

1. Use Case

    - You can production database that is taking normal load

    - You want to run a reporting app to run some analytics

    - You create a Read Replica to run a new workdload there

    - The reporting app can just do reads on the Read replica

        - READ only - SELECT, not INSERT, UPDATE or DELETE

    - prod db is unaffected

2. Network Cost

    - In AWS, theres a network cost when data goes from one AZ to another, but there are exceptions.

    - for RDS (managed service) Read Replicas within SAME REGION (different AZ, same region) - you do not pay that fee

    - for Cross Region replicas - there will be replication fee

### RDS Multi AZ

- Usually for disaster recovery

- SYNC Replication

- One DNS name - automatic app failover to standby

- increase availability

- failover in case of loss of AZ, loss of network, instance or storage failure

- no manual intervention

- NOT USED FOR SCALING

- however, Read Replicas can be setup as Multi AZ for Disaster Recovery (DR)

[App] <-- read and write --> [One DNS name - auto failover] --> [RDS MAster DB] -- sync replication --> [RDS DB instance standby]

EXAM: How to make an RDS DB go from Single AZ to Multi AZ

1. ZERO downtime operation

2. just click "modify" for the database

3. The following will happen:

    - A snapshot is taken

    - A new DB is restored from the snapshot in a new AZ

    - Synchronization is established between two databases

### Amazon Aurora

Writes are done via *Writer Endpoint* to reach the Writer replica. if writer instance go down, another Aurora instance promoted ad Writer instance.

Reads are done vai *Reader Endpoint* to react the Reader replica.

Global Aurora:

- Aurora Cross region Read Replicas:

    - helpful for Disaster Recovery

    - Simple to put in place

- Aurora Global Database (recommended)

    - 1 Primary Region (read/write)

    - Up to 5 secondary (read-only) regions, replication lag is less than 1 second

    - Up to 16 Read Replicas per secondary Region

    - Helps for decreasing latency

    - promoting another region (for disaster recover) has an RTO of < 1 minute

    - Typical cross-region replicated takes less than 1 second

What happens when we have Unplanned Failover:

1. We store Aurora endpoints in Parameter Store

2. App go through Parameter Store to get the Aurora Global endpoint

3. We need to have a lambda function to perform health checks against Aurora database

4. If it fails, it lambda will notify SNS topic, sends email to Admin

5. Admin will update Paramter Store with new Writer endpoint and Promote the Aurora from another Region as new Write Replica /  Primary Instance

Aurora Local and Global Mix Architecture:

1. In each region, we can have local Auora and global Aurora

2. Apps in same region can access local Aurora

3. Can also access Shared Datasets in Global Aurora - replication of Global Aurora between regions are done.

## 3.8 Amazon ElastiCache

Help you get managed Redis or Memcached.

Cache: in-memory databases with really high performance, low latency.

Help reduce load off of databses for read intensive workloads

Help make your apps stateless.

AWS takes care of OS maintenance/patching, optimizations, setup, configuration, monitoring, failure recovery, backups.

NOTE: ElastiCache involves heavy application code changes

1. Solution Architectures

- App queries ElastiCache, if not available (no cache hit/ cache miss), get from RDS, store in ElastiCache

    - Cache must have an invalidation strategy to make sure only the most current data is used in there.

- user logs into your app, app writes *session data* into ElastiCache.

- if user logs in another app (yours also), the instance retrieves the data and the user is already logged in

2. Redis vs Memchached

Redis:

- all about durability

- Can enable Multi AZ Replication and Auto-Failover

- Read  Replicas to scale reads and have high availability

- Data Durability using AOF persistence

- backup and restore features

- Supports sets and sorted sets

Memcache

- Multi-node for partitioning of data (sharding)

- No high availability / replication

- non persistent

- no backup and restore

- multi threaded architecture

3. Activity

- Go to ElastiCache > Redis > Create

- apps can use *Primary endpoint* or *Read endpoint* to reach the elastiCache

### ElastiCache Replication

2 Types:

1. Cluster Mode Disabled

- One primary node, up to 5 cache replicas

- if primary node fails, the replicas can take over

- asynchronus replication

- primary node for read and write

- replica nodes for read only

- ONE SHARD, all nodes have all the data

- guard against dat loss if node failure

- Multi AZ enabled by default for failover

- helpful to scale read performance

- 2 types of scaling:

    - Horizontal: scale out/in by adding/remove read replicas (max replicas)

    - Verical

        - Scale up/down to a larger/smaller node type

        - ElastiCache will internally create a new node group, then data replication and DNS update automatically

2. Cluster Mode Enabled

- Data is partitioned across shards

- helpful to scale writes

- each shard has a primary and up to 5 replica nodes (same)

- Multi AZ capability

- Up to 500 nodes per cluster

    - 500 shards with single master

    - 250 shards with 1 master and 1 replica

    - 83 shards with one master 5 replicas

### ElastiCache: Redis Auto Scale

Automatically increase/decrease the desired shards or replicas in your cluster

Supports both *Target Tracking and Scheduled Scaling Policies*

Works only For Redis with Custer Mode *Enabled*

eg.

- You have Redis ElastiCachse, with Cluster Mode enabled. Each shard has 1 primary node and 3 replicas.

- We track the *ElastiCachePrimaryEngineCPUUtilization* metric with CloudWatch.

- on CloudWatch, set *Target Tracking Policy*: *ElastiCachePrimaryEngineCPUUtilization = 60%*

- if yes, we trigger CloudWatch alarm, which can trigger increase desired count of Shard. Now you have 2 shards.

- Recommended: update apps to use the CLuster's *Configuration Endpoint*

1. ElastiCache Redis Connection Endpoints

- standalone Node

    - one endpoint for read and write operations

- Cluster Mode Disabled Cluster

    - Primary Endpoint - for al write operations

    - Reader Endpoint - evenly split read operations across all read replicas

    - Node Endpoint - for read operations

- CLuster Mode Enabled Cluster
    
    - Configuration Endppoint - for all read/write operations that support Cluster Mode Enabled commands

    - Node Endpoint - for read operations

## 3.9 Amazon Dynamo DB

Fully managed, highly available with replication across multiple AZ

IMPORTANT: NoSQL Database - not a relational database - with transaction support

Scale to massive workloads, distributed database

Can scale to millions of requests per second, trillions of row, 100s of TB of storage

- IMPORTANT: Fast and consistent performace (single digit millisecond)

Integrated with IAM for security, authorization and administration

Low cost and auto scaling capabilities

No maintenance or patching, always available

Standard and Infrequent Access (IA) Table Class.

1. Basics

- Dynamo DB is made of Tables

- Each table has a Primary Key (must be decided at creation time)

- Each table can have an infinite number of items / rows

- each has attributes (can be added over time - can be null)

- maximum size of an item is 400KB

- Data types supported:

    - Scalar - String, Number, Binary, Boolean, Null

    - DocumentTypes - List, Map

    - SetTypes - String Set, Number Set, Binary Set

- IMPORTANT: In Dynamo DB you can rapidly evolve schemas

2. Table Example:

[      Primary Key      ][     Attributes   ]
[Partition Key][Sort Key]
[   User_ID   ][ GAME_ID][ score ][ result  ]
[   123123    ][  1234  ][   15  ][  Win    ]

3. Read/Write Capacity Modes

- COntrol how to manage your table's capacity (read/write throughput)

- Provisioned Mode (default)

    - You specify the number of reads/writes per second

    - You need to plan capacity beforehand

    - pay for *provisioned* READ Capacity Units (RCU) & Write Capacity Units (WCU)

    - Possible to add *auto-scaling* mode for RCU and WCU

- On-Demand Mode

    - Read/writes automatically scale up/down with your workloads

    - No capacity planning needed

    - pay for what you use, more expensive

    - Great for *UNPREDICTABLE* workloads, *STEEP SUDDEN SPIKES*

4. Activity

- Go to DynamoDB > Create table

- Can use the capacity calculator

- View table > create item

### Dynamo DB: Advanced

1. DynamoDB Accelerator (DAX)

- Fully managed, highly scalable, seamless in-memory cache for Dynami DB

- Help solve read congestion by caching

- *Microseconds* latency for cached data

- DOES NOT require application code/logic modification - compatible with existing DynamoDB APIs

- 5 minutes TTL for cache (default)

2. DAX vs DynamoDB

- DAX:

    - infront of DynamoDb, helful for Individual objects cache & Query & Scan cache

- ElastiCache

    - If you wan tto store aggergation resul, ElastiCache is better

3. Stream Processing

- Ordered stream of item-level modifications (create/update/delete) in a table

- Use cases:

    - React to changes in real-time (welcome email to users)

    - real time usage analytics

    - insert into derivative tables

    - implement cross-region replication

    - invoke AWS lambda on changes to your DynamoDB table

- DynamoDB streams vs Kinesis Data Streams (newer)

    - DynamoDb steam:

        - 24 hour retention

        - limited no. of consumers

        - Process using AWS Lambda Triggers, or DynamoDB Stream Kinesis adapter

    - Kinesis Data Streams

        - 1 year retention

        - High no. consumers

        - Process using AWS Lambda, Kinesis Data Analytics, Kinesis Firehose, AWS Glue Streaming ETL


[App] ---CRUD---> [DynamoDB Table] which is either: 

    - [DynamoDb Streams] ---> Processing Layer:

        - [DynamoDB KCL Adapter] or [lambda] 
            
            ---message notif ---> Amazon SNS

            ---filter, transform ---> DDB Table
    
    - [Kinesis Data Streams] ---> [Kinesis Data Firehose] ---> Send to RedShift (for analytics), S3 (for archiving), OpenSearch (for indexing)

4. Global Tables

A table that is replicated across multiple regions.

2 way replication

Make DynamoDb table accessible with low latency in multiple regions

Active-active replication

    - applications can READ and WRITE to the table in any region

Must enable DynamoDB Streams as a pre requisite

5. Time to Live (TTL)

Automatically delete items after an expiry timestamp.

use case: reduce storage data by keeping only current items, adhere to regulatory obligations, web session handling...

6. Backup for disaster recovery

-  Continuous backups using point in time recovery (PITR)

    - optinally enabled for last 35 days

    - point-in time recover ty any time within backup window

    - the recovery process creates a new table

- On demand backups

    - full backups for long term terention until explicitely delted

    - doesnt affect performance or latency

    - can be configures and maanges in AWS Backup (enables cross region copy)

    - the recovery process creates a new table

7. DynamoDB and S3 Integration

    - Export table to S3 (must enable PITR)

        - works for any point of time in the last 35 days

        [DyanamoDb] ---export ---> [S3] <--- query --- [Amazon Athena]

        - does not affect read capacity of your table

        - perform data analysis on top of DynamoDB

        - retain snapshots for auditing

        - ETL on top of S3 data before importing back into DynamoDB

        - Export in DynamoDB JSON or ION format

    - Import from S3

        - Import CSV, DynamoDB JSON, or ION format

        - does not consume any write capacity

        - creates a new table

        - import errors are loggen in CLoudWatch Logs

## 3.10 Amazon DMS: Database Migration Service

Quickly and  securely migrate database to AWS

Resilient and Self healing

The source database remains available during migration

Supports:

1. Homogeneous migrations: ex Oracle to Oracle

2. Heterogeneous migrations: ex Microsoft SQL Server to Aurora

Continuous Data Replication using CDC

You must create an EC2 instance to perform the replication tasks.

[Source DB] ---> [EC2 with DMS + SCT (optional)] ---> [Target Db]

1. Sources:

- On Premise and EC2 instances databases: Oracle MS, SQL Server, MySQL, MariaDB, PostgreSQL, MongoDB, SAP, DB2

- Azure: Azure SQL Database

- Amazon: All including Aurora

- S3

- Document DB

2. Targets

- On Premise and EC2 instances databases: Oracle MS, SQL Server, MySQL, MariaDB, PostgreSQL, MongoDB, SAP, DB2

- Amazon RDS

- Redshift, Dynamo DB, S3

- OpenSearch Service

- Kafka

3. Activity

- Go to DMS > Replication Instances > Create Replication Instance

- Create Data Migration Task, reference Replication Instance in prev step

If source Db and Target does not have the same Engine: need to use AWS Schema Conversion Tool (SCT):

### AWS Schema Conversion Tool (SCT)

Converts your Database Schema from one engine to another.

Example OLTP: (SQL Server or Oracle) to MySQL, PostgreSQL, Aurora, etc

Example OLAP: (teradata or Oracle) to Amazon Redshift

IMPORTANT: You DO NOT NEED to use SCT if you ar emigrating the same DB engine.

Eg. On Premise PostgreSQL ---> RDS PostgreSQL

### DMS: Continuous Replication

See diagram

### DMS: Multi AZ Deployment

When multi AZ Enabled, DMS provisions and maintians a synchronously stand replica in a different AZ (standby replica)

- Provides Data Redundancy

- Eliminate IO Freezes

- Minmize latency spikes

### DMS: Replication Task Monitoring

- Task Status - Creating, Running or Stopped

    - task status bar

- Table State

    - includes the current state of the tables (eg. BeforeLoad, TableCompleted...)

    - number of inserts, deletion, and updates for each table

- CloudWatch metrics

    - Replication Task metrics

        - Statistics for Replication Task including incoming and committed changes, latency between the Replication Host and both source and target datavases

        - FullLoadThroughputRowSource, FullLoadThroughputRowsTarget, CDCThroughputRowSource, CSCThorughputRowsTarget - CDC - Continunious Replication

    - Host metrics

        - represents performance and utilization statistics for the replicatio host

        - CPUUtilization, FreeableMemory, FreeStorageSpace, WriteIOPS

    - Table metircs

        - Statistics for tables that are int he process of being migrated, including the number of insert, update, delete, and DDL Statements completed

## 3.11 S3 Replication

When we have 1 S3 bucket in one region, and have a target bucket in another region.

We want to setup ASYNCHRONUS replication between the two buckets.

Buckets can be in different AWS accounts.

2 Flavors:

1. CRR (Cross Region Replication) - Regions must be different

2. SRR (Same Region Replication) - Regions must be same

To do this.

1. Must enable Versioning in source and destination buckets

2. Must give proper IAM permissions to S3

Use Cases:

- CRR:

    - compliance, lower latency access, replication across accounts

- SRR: 

    - log aggregation, live replication between production and test accounts

Activity:

- Go to S3 > Create a new bucket > set the region > enable versioning

- Create another target bucket > set the region > enable versioning

- Go to source bucket > management > replication rules > Create replication rule  > specify destination bucket

IMPORTANT: it will only enable replication from the moment you set it- ie. for objects created after the time you enable the replication. If you wan tto replicate the previous objects, use *S3 Batch Operations*


## 3.12 Amazon Storage Gateway

Hybrid Cloud for Storage

- part of your infra is on premises

- part of your infra is on the cloud

Can be due to:

- Long cloud migrations

- security requirements

- compliance requirements

- IT strategy

S3 is a proprietary storage technology (unline EFS/NFS), so how do you expose the S3 data on premise? -  AWS Storage Gateway

AWS Storage Cloud Native Options:

1. Block 

    - EBS

    - EC2 instance Store

2. File

    - Amazon EFS

3. Object

    - S3

    - Glacier

AWS Storage Gateway bridges between on-premise data and cloud data in S3

Hybrid storage service to allow on-premise to seamlessly use the AWS Cloud

Use cases: disaster recovery, backup and restore, tiered storage

Type of Storage Gateway:

1. File Gateway

2. Volume Gateway

3. Tape Gateway

### File Gateway: Deep Dive

1. RefreshCache API

Storage Gateway updates the File Share Cache automatically when you write files to the File Gateway.

When you upload files directly to the S3 bucket, users connected to the File Gateway may not see the files on the File share (accessing stale data) - YOu need to invoke *RefreshCache API*. This can be done:

- on demand, or;

- using Periodical Lambda function; or

- triggering it automatically from S3 through lambda - but will be expensive if you have many items in the S3

2. Automating Cache Refresh

File Gateway feature that enables you to automatically refresh File Gateway to stay up to date with the changes in their S3 buckets

Ensures user are not accessing state data on their file shares

No need to manually or periodically invoke the RefreshCacheAPI

## 3.13 Auto Scaling Groups (ASG)

### Scaling Policies

1. Dynamic Scaling

    - Target Tracking Scaling

        - Simple to set up

        - eg. I want the average ASG CPU to stay around 40%

        - will trigger CloudWatch Alarm

    - SImple / Step scaling

        - Based on Alarm
        
        - when CloudWatch alarm is triggered (example CPU > 70%), then add 2 units

        - when CloudWatch alarm is triggered (example CPU < 30 %) then remove 1

    - Scheduled Scaling

        - anticipate a scaling based on known usage patterns

        - eg. increase the min capacity to 10 at 5pm on Fridays

    - Predictive Scaling: continuously forecast load and schedule scaling ahead

2. Good metrics to scale on:

    - CPUUtilization: Average CPU utilization across your instances

    - RequestCountPerTarget: to make sure the number of requests per EC2 instances is stable

    - Average Network IN/Out (if youre app is network bound)

    - Any custom metric (that you push to CloudWatch)

3. Scaling Cooldown

    - After a scaling activity happens, you are in the cooldown period (default 300 seocnds)

    - During the cooldown period the ASG will not launch or teminate additional instances (to allow for metrics to stabilize)

    - Advice: user a ready-to-use AMI to reduce configuraiton time in order to be serving request faster and reduce the cooldown period

4. Activity

    - Go to EC2 > Auto Scaling Groups > Select your ASG

    - In the Automatic Scaling tab > Create Scheduled Scaling Policy

    - Select Metric

    - Using AWS Console, connect to an instance

    - install *stress Aamzon linux 2*

    ```
    sudo amazon-linux-extras install epel -y

    sudo yum install stress -y
    ```

    ```
    stress -c 4 # make cpu go 100%
    ```

    - Go to  CloudWatch > Alarms > alarm should be triggered


### ASG Lifecycle Hooks

By Default, as soon as an instance is launched, in an ASG its in service

You can perform extra steps before the instance goes in service (*Pending state*) - Pending:Wait, Pending Proceed

    - define a script to run on the instances as they start

You can perform some action sbefore the nstance is terminated (*Terminating state*) - Terminating:Wait, Terminating Proceed

    - pause the instances before theyre terminated for troubleshoooting

Use cases: cleanup, log extraction, special health checks.

Integration with EventBridge, SNS and SQS

1. Example:

[ASG with Instance A, B C]

Instance C terminates and goes into *Terminating:Wait* + termination Event sent to EventBridge

EventBridge invokes System Manager's Run Commond to collect logs from Instance C using hook

### ASG SNS Notifications

ASG Supports sending SNS notifications for the following events:

1. autoscaling:EC2_INSTANCE_LAUNCH

2. autoscaling:EC2_INSTANCE_LAUNCH_ERROR

3. autoscaling:EC2_INSTANCE_TERMINATE

4. autoscaling:EC2_INSTANCE_TERMINATE_ERROR

Another better way: Use EventBridge:

- can crewate Rules that match the followwing ASG events:

    - EC2 Instance launching, EC2 Instance Launch Successful/Unsuccessful

    - EC2 Instance Terminating, EC2 Instance Terminate Succesful/Unseccuessful

    - EC2 Auto Scaling Instance Refresh Checkpoint Reached

    - EC2 Auto Scaling Instance Refresh Started, Succeeded, Failed, Cancelled

### ASG Termination Policies

Determine which instances to terminate first during scale-in events, Instance Refresh, and AZ Rebalancing

Default Termination Policy:

- Select AZ with more instances

- Terminate instance with oldest *Launch Template* or *Launch Configuration*

- If instances were launched using the same *Launch Template*, temrinate the instance that is closest to the next billing hour

1. Different Termination Policies:

- default: terminates instances according to Default Termination Policy

- Allocation Strategy - terminates instances to align the remaining instances to the Allocation Strategy (eg. lowest-price for Spot instances, or lower priority ON-Demand Instances)

- OlderstLaunchTemplate - terminates instances that have the oldest Launch Template

- OldestLaunchConfiguration - terminates instances that have the olderst Launch Configuration

- ClosestToNextInstanceHour - terminates instances that are closest tot he enxt billing hour

- NewestInstance - temrinate the newest instance (testing new launch template)

- OldestInstance - terminates the oldest instances (upgrading instance size, not launch template)

NOTE: you can use one or more policies and specify the evaluation order

NOTE: you can define Custom Termination Policy, backed by Lambda Funtion

### ASG Warm Pools

When an ASG scales out, it tries to launch instances as fast as possible

Some apps contain lengthy unavoidable latency that exits at the application initialization / bootstrap layer (several minutes or more)

Processes that can only happen at initial boot: applying updates, data or state hydration, runnign configuration scripts...

Solution was to over-provision compute resources to absorb unexpected demand increases (increases cost) or use Golden Images to try to reduce boot time.

New solution: use ASG Warm Pools

1. Reduces scale-out latency by maintaing a pool of *pre initialized* instances

    - In a scale out event, ASG uses the pre-initialized instances from the Warm Pool instead of launching new instances

    - Warm Pool Size Settings:

        - Minimum warm pool size (always in the warm pool)

        - Max prepared capacity - set number of instances

    - Warm Pool instance state - what state to keep your Warm Pool instances in after initialization (Running, Stopped, Hibernated)

    - Warm Pools instances dont contribute to ASG metrics that aaffect Scaling Policies

2. Warm Pools pricing: m5.large

    - if we "over provision an EC2 instance":

        Running Cost = $0.096/hour * 24 hours/day * 30 days = $69.12 + EBS Charges

    - if the EC2 instance is stopped, we only pay for the attached EBS volume

    - EBS Volume Cost = $0.10/GB-month * 10GB = $1.00

3. Running vs Stopped vs Hivernated:

- Running:

    - Scale Out Delay : fastest (immediately available to accept traffic)

    - Start Up Delay : Lower (EC2 instances already running)

    - Cost - higher (incur cost for usage even when not serving traffic)

- Stopped

    - Scale Out Delay : slower (needs to be restarted before traffic can be served)

    - Start Up Delay : Slower (Needs to go through app startup process)

    - Cost - lower (do not incur running cost, only par for attached resources)

- Hibernated

    - Scale Out Delay : Medium (pre- initialized EC2)

    - Start Up Delay : Medium (faster than in the stopped state)

    - Cost - Lower (do not incur running cost, only par for attached resources)

        - the hibernation state must be able to fir onto the EBS volume (might need a bigger volume)

4. Warm Pools Instance Reuse Policy

- By default, ASG terminates instances when AsG scales in, then it launches new instances into the Warm Pool

- Instance Reuse Policy allows you to return instances to the Warm Pool when scale in event happens

5. WArm Pools Lifecycle Hooks

[start] --> Warmed:Pending ---> Wramed:Pending:wait ---> warmed: Pending:Proceed ---> Warmed:Stopped or Running or Hivernated ---> [scale out] ---> Pending ---> Pending:wait ---> Pending:Proceed ---> InService ---> [Scale In ]---> Terminating --- > Terminating:Wait ---> Terminating Proceed --- > Terminated ---> End

## 3.14 AWS Application Auto Scaling

Monitors your apps and automatically adjusts capacity to maintain steady, predictable performance at lowest cost.

Setup scaling for multiple resources across multiple services from a single place (no need to navigate acriss different services)

Point to your app and select the services and resources you want to scale (no neeed to setup alarms and scaling actions for each service)

Search for resources/services using CloudFormation Stack, Tags or EC2 ASG

Build *Scaling Plans* to automatically add/remove capacity from your resources in real-time as demand changes

Suppports *Target Tracking, Step and Scheduled Scaling Policies*

## 3.15 Elastic Load Balancer (ALB) Rules

1. Listener Rules

    - processed in order (with Default Rule)

    - each rule supports Actions (forward, redirect, fixed-response)

    - Rule conditions:

        - host-header

        - http-request-method

        - path-pattern

        - source-up

        - http-header

        - query string

2. Target Group Wighting

- specify weight for each Target Group on a Single Rule

- Example: multiple versions of your app, blue/green deployment

- allows you to  control the distribution of the traffic to your apps

[Users] ----> ALB with Rules (Target Group 1: 8, Target Group 2: 2) --- direct requests based on weight to respective groups

3. DualStack Networking

- Allows clients communicate with the ELB using both IPv4 and IPv6

- supports both ALB and NLB

- ALB and NLB can have mixed IPv4 and IPv6 targets in seperate target groups

- ELB DualStack ensures compatibility between client and target IP versions

    - IPv4 clients comm with IPv4 targets, and Ipv6 clients comm with IPv6 targets

    - if you only have IPv4 targets. the ELB automatically converts requersts from IPv6 to IPv4

Note: Az must be added/enabled for instacnes to receive traffic

4. NLB PrivateLink Integration

Exposing Service in VPCs with overlapping IP addresses (instead of VPC Peering)

## 3.16 AWS NAT Gateway

AWS-managed NAT, higher bandwidth, high availbility, no administration

Pay per hour for usage and bandwidth

NATGW is created ina specific AZ, used an Elastic IP

CANNOT BE USED by EC2 instance in the same subnet (only from other subnets)

Requires an IGW (Private Subnet => NATGW => IGW (internet gateway))

5Gbps of bandwidth with automatic scaling up to 100 Gbps

No Security Groups to manage/required

1. NATGW with High Availability

- NATGW is resilient ONLY within a single Availability Zone

- Must create multiple NATGWs in *MULTIPLE AZs*for fault tolerance

2. NAT Gateway vs NAT Instance

- NAT Gateway:

    - Highly available within AZ

    - bandwidth up to 100Gbps

    - Maintenance managed by AWS

    - Cost Per hour & amount of data transferred

    - Supports Public and Private IPv4

    - NO Security Groups

    - CANNOT use as Bastion Host

- NAT Instance

    - need to use a script to manage failover between instances

    - Bandwidth depends on EC2 instance type

    - Maintenance managed by you (software updates, OS patches..)

    - Cost per hour, EC2 instance type and size + network costs

    - Supports Public and Private IPv4

    - Supports Security Groups

    - CAN be used as Bastion Host

## 3.17 Multi AZ Architectures

1. Services where Multi-AZ must be enabled manually:

- EFS, ELB, ASG, Beanstalk: assign/ choose which AZ to deploy in

- RDS, ElastiCache: multi AZ (synchronous standby DB for failovers)

- Aurora

    - data is tored in automatically across multi AZs

    - can have multi AZ for the DB itself (same as RDS)

- OpenSearch (managed): multi master

- Jenkins (self deployed): multi master


2. Services where Multi-AZ is implicitly there:

- S3 (except OneZone-Infrequesnt Access)

- DynamoDB

- All of AWS' proprietaty, managed services

3. Basic architectures:

- One big VPC, containing:

    - Availability Zone A

    - Availability Zone B

- Each AZ

    - 1 Public Subnet for ALB

        - requests are spread out across multiple AZs, to the ALBs

    - 1 Private Subnet for Application Tier

    - 1 Private Subnet for Data Tier

        - Private Subnet for AZ A: RDS DB (with MULTI AZ) as Main

        - Private Subnet for AZ B: Secondary RDS instance for standby

- ALB Sends requests to Target Group: Application Tier, spread across the AZs

- Requests are forwarded to Main RDS instance in AZ A

## 3.18 Blue Green Architectures

1. Arch 1:

- 1 ALB as Listener, spreading requests across Target Groups

- 2 Target Groups

    - Blue Target Group

    - Green Target Group

- Switch Traffic *at Once* or use *weighted TG*

2. Arch 2:

- 2 ALB as Listener, each pointing to one Target Group

- 2 Target Groups

    - Blue Target Group

    - Green Target Group

- Route 53 (as DNS) points to ALB 1 and ALB 2

    - traffic can be switched using DNS

    - switch is not immediate (All at once not possible)

3. Arch 3: If you use API Gateway: Option 1

- 1 API Gateway (Prod Stage)

- API Gateway connected to Lambda v1

- Client sends requests to that API Gateway Stage

- We can create Another stage (Prod Stage Canary)

- Prod Stage Canary points to Lambda v2

- Clients are redirected to Prod Statge Canary

- Can switch all at one or gradually

4. Arch 4: If you use API Gateway: Option 2

- 1 API Gateway (Prod Stage)

- API Gateway connected to Lambda *Alias* (Alias: Prod)

- Alias points to Lambda v1

- Client sends requests to that API Gateway Stage

- We can deploy Lambda v2

- Control traffic switching from the Alias

- no changes required to the API Gateway and Client

### 3.19 Multi Region Architectures

1. With Route 53

- by doing Health check => automated DNS failovers

- different kinds of health checks:

    - Health checks that monitor an endpoint (application, server, other AWS resource)

        - helpful to create `/health` route

    - Health Checks that monitor other health checks (calculated health checks)

    - (important) Health Checks that monitor CloudWatch alarms (full control!!)

        eg. throttles of DynamoDB, custom metrics, etc

    - Health checks are integrated with CloudWatch metrics

- Basic arch:

    - Route 53 points to the ALBs 2 Regions

    - each Region has ALB, ASG instances

    - Health checks done by Route 53 record when calling the ALBs

2. Another way 1:

- we have 2 regions, each with 1 ALB, pointing the their respective Target Groups

- Route 53 can be used to do Latency-based, to redirect to ALB of the Region closest to the user

3. Another way 2:

- we have 2 regions, each with 1 API Gateway, pointing to their respective 1 Lambda

- Route 53 is used as Latency-based, to redirect user requests to API Gateway in closest region

- lambda in Region A connects to DynamoDB as Global Table, which will be replicated across to Regon B As well

- Each lambda from each region will benefit Low Latency data access to Dynamo DB

4. Another way 3: Complex Routing

- we have 2 regions, each with 1 ALB

- Setup Route 53 record (us.example.com) as a *Failover Routing*. Primary points to ALB of Region A and Secondary points to ALB of Region B

- Setup Route 53 record (eu.example.com) as a *Failover Routing*. Primary points to ALB of Region B and Secondary points to ALB of Region A

- Setup Route 53 (example.com) to have Latency-Based Routing as aliases to us.example.com and eu.example.com

## 3.20 Disaster Recovery in AWS

Disaster - an event that has negative impact on a company's business continuity or finances is a disaster

Involves PREPARING for and RECOVERING from a disaster

Types:

1. On Premise => disaster => On Premise (traditional DR, very expensive)

2. On Premise => disaster => AWS Cloud (hybrid)

3. AWS Cloud Region A => AWS Cloud Region B

Terms:

1. RPO: Recovery Point Objective

- How often do you run backups

- Time between Disaster and RPO backup => data loss

2. RTO: Recovery Time Objective

- When you recover from Disaster

- time between Disaster and RTO => Downtime

Disaster Recovery Strategies:

1. Baackup and Restore

- Slowest RTO ie. High RTO

- Easy and Cheap

- High RPO

- Methods:

    - [corporate data center] --> [AWS Storage Gateway] --> Amazon S3 + Lifecycle Policy to move data to Glacier 

    OR

    - [corporate data center] -- once a week --> [AWS Snowball] --> Glacier

    OR

    - [AWS Cloud / EBS/ RedShift / RDS] -- schedule regular snapshots --> Snapshot

- Restoration:

    - from AWS (Glacier or Snapshot)

    - Use AMIs to recreate EC2 instances

    - restore from Snapshot

2. Pilot Light

- A small version of the app is always running in the cloud

- Useful for the CRITICAL CORE

- very similar to Backup and Restore

- Faster than BnR as critical systems are already up

- Methods:

    - [corporate data center] --- data replication ---> AWS RDS (running) + EC2 not running

    - Route 53 points to Corporate Data Center and EC2 as failover

    - if disaster strikes, Route 53 redirect to the EC2, and make it up and running + RDS already running

- Faster RTO ie. Lower RTO

3. Warm Standby

- Full system is up and running but minimum size

- upon disaster, we scale to production load

- method:

    - corporate data center has [reverse proxy] to [App Server] which points to [Master DB]

    - Route 53 points to [corporate data center] and [ELB] of AWS Cloud

    - on AWS Cloud, we have RDS Slave (running), with regular Data Replication from Master DB

        - we also have EC2 Auto Scaling (minimum), points to Master DB

        - and ELB ready to go

    - when disaster strikes, Route 53 moves traffic to [AWS ELB]

        - EC2 then points to RDS

- More expensive

- Faster than Pilot Light RTO ie. Low RTO

4. Hot Site / Multi Site Approach

- Fastest RTO ie. Lowest RTO

- Very Expensive

- Same as Warm Standby, but ES2 Auto Scale is running at Production scale

5. All AWS Multi Region

- Cloud 1

    - ELB

    - EC2 Auto Scaling (production)

    - Aurora Global (master), with Data replication to Aurora Global (slave) of Cloud 2

- Cloud 2

    - ELB

    - EC2 Auto Scaling (production), talks to Aurora master at Cloud 1, but failover to Aurora Global Slave

    - Aurora Global (slave)

- Route 53 Points to ELBs

Tips:

1. Backup

- EBS Snapshots, RDS automated backups, Snapshots, etc..

- Regular pushes to S3/S3 IA/Glacier, Lifecycle Policy, Cross Region Replication

- From On-Premise: Snowball or Storage Gateway

2. high Availablility

- use Route 53 to migrate DNS over from Region to Region

- RDS Multi AZ, ElastiCashe Multi Az, EFs, S3

- Site to Site VPN as a revovery from Direct Connect

3. Replication

- RDS Replication (Cross Region), AWS Aurora + Global Databases

- Database replication from on-premise to RDS

- Strorage Gateway

4. Automation

- CloudFormation / Elastic Beanstalk to re create a whole new environment

- reover/ reboot EC2 instances with CloudWatch if alarms fail

- AWS lambda functions for cuistomized automations

5. Chaos

- Neflix has "simian-army" randomly terminating EC2