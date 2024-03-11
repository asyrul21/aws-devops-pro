# Domain 2: Configuration Management and Infra as Code: Part 2

## 2.2 Service Catalog

For governance, compliance, and organizational consistency.

User that are new to AWS may have too many options, may create stacks that are not compliant/in line with the rest of the organization

Some users just want a quick *self-service* portal to launch a set of authorized products pre-defined by admins.

AWS Service Catalog: Giving users a Controlled environment to deploy stuff to AWS. A Set of CloudFormation templates that users can use based on their IAM permission.

Service Catalog integrates with "self-service portals". Eg. ServiceNow

1. Admin Tasks

    - Define a Product. Product is a CloudFormation Template

    - Portfolio: A Collection of Products.

    - IAM Permissions to access Each Porfolio

2. Users

    - will be presented a Product List, each authorized by IAM

    - launch these products

    - Provisioned Products are ready to use, properly configured

EXAM SCENARIO:

- can give user access to *Launching Products* without requiring deep AWS Knowledge. Define CloudFormation templates in advance, then use Service Catalog for that user to launch

### Extras

1. Stack Set Constraints

- Allow you to configur Product deployment options using CloudFormation using StackSets

Can restrict by:

    - Accounts - identify AWS accounts where you want to create Products

    - Regions - identify AWS regions where you want to deploy to (with Order)

    - Permissions - IAM StackSet Administrator Role to manage target AWS accounts

2. Launch Contrianst

    - IAM Role assigned to Product which allows a user to launch, update, or terminate a product with minimal IAM permissions

    - Example: end user has access only to Service Catalog, all other permissions required are attached to the Launch Contraint IAM Role

        - IAM role must have the following permissions:

            - CloudFormation (FullAccess)

            - AWS Services in the CloudFormation template

            - S3 Bucker which contains the CloudFormation template (Read Access)

3. CI/CD - Syncing with CodeCommit

    - developer pushes to CodeCommit. Must have `mapping.yml`.

        - `mapping.yml` specifies which template files are for which Product

    - invoke Lambda function, which update/create Product in Service Catalog

## 2.3 Elastic Beanstalk

A Developer-Centric approach of deploying an application on AWS.

Uses all the common services: EC2, ASG, ELB, RDS, without having the developer to configure these manually.

Managed Service:

    - Automatically handles capacity provisioning, load balancing, scaling, app health monitoring, instance configs,...

    - We still have full control over the configuration

    - Beanstalk is FREE but you pay for the underlying instances

### EBS: Components

1. Application: collection of EBS components (environments, version, configs)

2. Application Version: an iteration of your application code

3. Environment:

    - Collection of AWS resources running an application version (one version at a time)

    - Tiers: 
    
        - Web Server Environment Tier (ELB + EC2)

            - usually accepts client requests > ELB > ASG > EC2 instances as web servers

            - if there are long processing requests, send it to a Worker Tier SQS Queue (decouple)
        
        - Worker Environment Tier (SQS + EC2)

            - does not deal with client requests

            - deals with SQS Queue (Messaging Queue) > ASG > EC2 Instance as Worker (pulls messages from the SQS Queue)

            - scale base on the number of SQS messages

            - can push messages to SQS queue from another Web Server Tier

    - can create multiple environments (dev, test, prod)

4. Deployment Modes:

    - Single Instance: Great for Dev

        - Availability Zone -> Elastic IP -> EC2 instance + RDS Master

    - High Availability with Load Balancer

        - Great for Prod

        - ALB -> ASG -> Availability Zone -> EC2 + RDS master
                     -> Availability Zone -> EC2 + RDS Standby

### EBS: Deployment Options for Updates

1. All at once 

    - deploy all in one go 
    
    - fastest, but there will be some downtime

    - apps on all instances are stopped

    - no additional cost

2. Rolling 
    
    - update a few instances at a time (bucket) then move onto the next bucket once the first is healthy

    - application is running below capacity - based on number of "bucket"

    - eg. 2 out 4 is stopped, updated. If ok, move on the next bucket.

    - no additional cost, long deployment

3. Rolling with additional batches

    - Zero downtime

    - like rolling, but spins up new instances to move the batch (so that the old application in still available)

    - instances are running at full capacity for eg (4/4)

    - can set bucket size. For eg, bucket size = 2, there will be 2 additoinal instances added. Total now: 6 instances. Once done, the next bucket size (eg 2) instances out of the 6 will be stopped and updated.. and so on

    - our app runs both versions simulataneously

    - small additional cost

    - additional batch is removed at the end

    - long deployment

    - Good for prod

4. Immutable: 

    - Spins up new nstances in a new ASG, deploy version to these instances, then swaps all the instances when everything in healthy

    - Zero downtime

    - High cost, double capacity

    - Longest deployment

    - Quick rollback in case of failures - just terminate the new ASG

    - Great for prod

5. Blue Green - involves *Route 53*

    - not EBS feature, proces is quite manual

    - Crea a new environment and switch over when ready

    - Zero downtime, helps with release facility

    - Create a new "stage" environment and deploy v2 there

    - the new env (green) can be validated and roll back if issues

    - can use Route 53 with weighted policies to redirect a little bit of traffic to the stage environment

    - on EBS, you can "swap URLs" when done with environment test

6. Traffic Splitting - Involves *ASG / ALB*

    - Canary testing - send small % of traffic to new deployment

    - New application version is deployed to a temporary ASG with same capacity

    - small % of traffic is sent to the temporary ASG for configurable amount of time

    - deployment health is monitored

    - if theres a deployment failure, thi striggers an automated rollback (very quick)

    - no downtime

    - new instances are migrated from the temporary to original ASG

    - old app version terminated

- Activty:

    1. EBS > Environement > Go to your environment > Configuration

    2. Under *Rolling Updates and Deployments* > Deployment Policy > Choose Deployment modes

    3. Under Actions > Swap environment domain - prod becomes dev, dev becomes prod

### EBS: Extras

1. Web Server Deployment vs Worker Deployment

    - if your application perform tasks that are long to complete, offload these tasks to a dedicated worker environment- decoupling

    - define periodic task in `cron.yml`

    - eg. process a video, generate a zip file

2. Notifications with EventBridge

    - EBS emits those events to EventBridge

    - Create Rules in EventBridge to act to the following events:

        - Environmnet Operation Status - create, update, terminate (start, success, fail)

        - Other resources' status - SG, ALB, EC2 instance (created, deleted)

        - Managed Updates Status - started, failed

        - Environment Health Status

    - you can:

        - trigger SNS to send email to admin

        - invoke Lambda to for eg. send a message to Slack


## 2.4 Serverless Application Model (SAM)

Framework for developing and deploying serverless applications.

All configuration in YAML code.

Generate complex CouldFormation from simple SAM YAML file

Supports anything from CloudFormation: Outputs, Mappings, Parameters, Resources

Only two commands to deploy to AWS

Can integrate with CodeDeploy to deploy Lambda Functions.

Can help run Lambda, API Gateway, DynamoDB locally

### AWS SAM: Recipe

1. Transform Header indicates its a SAM template:

```yml
Transform: 'AWS::Serverless-2016-10-31'
```

2. Write Code

- `AWS::Serverless::Function` : lambda

- `AWS::Serverless::Api` : Api Gateway

- `AWS::Serverless::SimpleTable` : DynamoDB Table

3. Build

- `sam build`

4. Package and Deploy commands

- `aws cloudformation package /` or `sam package`

- `aws cloudformation deploy` or `sam deploy`

Sam Template (yml) + App Code ---> `sam build` to transform to a CloudFormation template + App Code ---> `sam package` to package the app and send to S3 bucket ---> `sam deploy` to create/execute ChangeSet in CloudFormation ---> CloudFormation with then deploy the CloudFormation Stack

### AWS SAM: Running Lambda Locally / CLI debugging

1. Use the SAM CLI to run Lambda locally

    - provides a lambda-like execution environment locally

    - SAM CLI + AWSToolkits => step-through code and debug

    - Toolkits include: AWS Cloud9, VSCode, JEt5Brains, PyCharm, IntelliJ

    - the Toolkits are IDE plugins which allows you to invoke Lambda functions locally with SAM

### AWS SAM: Integration with CodeDeploy

1. SAM Framework natively uses CodeDeploy to update Lambda functions

    - traffic shifting feature from v1 to v2 within a Lambda Alias

    - pre and post traffic hooks to validate deployment

    - easy and automated rollback using CloudWatch Alarms

```yml
Resources:
    MyLambdaFunction:
        Type: AWS::Serverless::Function
        Properties:
            Handler: index.handler
            Runtime: nodejs12.x
            CodeUri: s3://bucket/code.zip

        # HERE
        AutoPublishAlias: live

        # HERE
        Deployment Preference:
            Type: Canary10Percent10Minutes
            Alarms:
                # A list of alarms tha tyou want to monitor
                - !Ref AliasErrorMetricGreaterThanZeroAlarm
                - !Ref LatestVersionErrorMetricGreaterThanZeroAlarm
            Hooks:
                # Validation lambda functions that are run before and after traffic shifting
                PreTraffic: !Ref PreTrafficLambdaFunction
                PostTraffic: !Ref PostTrafficLambdaFunction
```

- the *AutoPublishAlias*

    - detects when new code is being deployed

    - creates and publishes and updated version of that function with the latest code

    - points the alias to the updated version of the Lambda function

- *Deployment Preference*:

    - Canary, Linear, AllAtOnce - same with CodeDeploy

- *Alarms*

    - Alarms that can trigger rollback

- *Hooks*

    - Pre and post traffic  shifting Lambda functions to test your deployment

## 2.4 AWS Cloud Development Kit (CDK)

Define cloud infrastructure using a familiar programming language: Javascript/Typescript, Python, Java, .NET etc.

Contains high level components called `constructs`.

Can leverage compilation process to check for syntax errors, etc.

The code is compiled into a CloudFormation template (json/yml).

Can deploy infrastructure and application runtime code together:

- great for Lambda functions

- great for Docker containers (ECS/EKS)

### CDK vs SAM

SAM:

    - Serverless focused

    - write your template declaratively in JSON or YAML

    - great for quickly getting started with Lambda

    - Leverages CloudFormation

CDK

    - All AWS Services

    - write infra in Programming Language

    - also leverages CloudFormation

Combining:

    - Use SAM CLI to test your CDK apps

    - YOu must first run `cdk synth`

    - CDK APP (Lambda Function: eg. `myFunction`) -> cdk synth -> CloudFormation Template (eg `myCDKStack.template.json`) -> SAM CLI (`sam local invoke -t myCDKStack.template.json myFunction`)

### Activity

1. Open AWS CloudShell

```bash
# 1. install the CDK
sudo npm install -g aws-cdk

# create app directory
mkdir cdk-app
cd cdk-app/

# initialize the CDK application
cdk init --language javascript

# verify it works correctly
cdk ls

# install the necessary packages
npm install @aws-cdk/aws-s3 @aws-cdk/aws-iam @aws-cdk/aws-lambda @aws-cdk/aws-lambda-event-sources @aws-cdk/aws-dynamodb


# 2. copy the content of cdk-app-stack.js into lib/cdk-app-stack.js


# 3. setup the Lambda function
mkdir lambda && touch index.py

# 4. bootstrap the CDK application # run once per account per region
# This will create a CDKToolkit Stack in your CloudFormation
cdk bootstrap

# 5. (optional) synthesize as a CloudFormation template
# ie. compile your code to a CloudFormation template
cdk synth


# 6. deploy the CDK stack
# send the CloudFormation template to CloudFormation
cdk deploy

# (optional)
# 7. empty the s3 bucket
# 8. destroy the stack
cdk destroy
```

## 2.5 AWS Step Functions

Model your workflow as State Machine (one per workflow), in JSON.

Visualization of the workflow and the execution of the workflow, and their history.

Start workflow with SDK call, API Gateway, Event Bridge (CloudWatch Event).

1. Task States

    - Do some work in your state

    - invoke ONE AWS Service

        - Can invoke a Lambda function

        - Run an AWS Batch job

        - Run an ECS task and wait for it to complete

        - etc

    - OR Run one Activity

        - Actiities send results back to Step Function

Example - Invoke a Lambda Function

    - see `step-functions/state-machine.json`

### Step Function States

1. Choice / Condition

2. Failed or Succeed state - stop axecution with failure or success

3. Pass

4. Wait - delay

5. Map State - Dynamically iterate steps

6 (IMPRTANT EXAM) Parallel State - begin parallel branches of execution

### Activity

1. Go to AWS Step Function > Create State Machine

    - execution role is created automatically

## 2.6 AWS AppConfig

Deploy, validate, and deploy dynamic configurations.

App configurations are placed external (thus independent) from app code.

Do not need to restart the application.

Enables:

    - feature flags

    - appplication tuning

    - allow/block listing

Use with various AWS services/apps: EC2, Lambda, ECS, EKS, etc

    - Apps depending on the configuration will regularly *Poll* fir config changes

    - Gradually deploy the configuration changes and rollback if issues occur

    - if things go wrong, it will trigger CloudWatch alarm

Validate configuration changes before deployment using:

- JSON Schema (for syntactic check)

- Lambda function - run code to perform validation (semantic check)

## 2.7 AWS Systems Manager (Exam Question)

Helps you manage your EC2 and on-Premise systems at scale.

Get oprational insights about the state of your infra.

Easily detect problems

*Patching automation for enhanced compliance*

Integrated with:

- CloudWatch metrics and dashboards

- AWS Config

Free.

### Basic Features

1. Resource Groups

2. Operations Management

3. Shared Resources

    - Documents

4. Change Managements

    - Automation

    - maintenance Windows

5. Application Management

    - Parameter Store

6. Node Management

    - Inventory

    - Session Manager

    - Run Command

    - State Manager

    - Patch Manager

### SSM: how it works

1. install SSM Agent onto the system we control

    - installed by default on Amazon Linux 2 AMI and some Ubuntu AMI

2. If instance cannot be controlled with SSM, its probably issues with the SSM Agent or IAM Permissions

    - Make sure the EC2 instances have a proper IAM role to allow SSM Actions

### SSM: Activity

1. In EC2 > Launch Instance > Choose Amazon Linux (SSM Agent pre installed and setup)

    - Under Network Settings > dont need to allow any connectivity: no ssh, no https - we access via SSM Agent

    - Create new IAM Instance Profile > Create Role > For EC2 > Add Policies:

        - AmazonSSMManagedInstanceCore

1. Go to SSM > Fleet Manager > You can see the instances you created in Previous step.

## 2.8 AWS Tags

You can add key-value pairs called Tags to many AWS resources

Commonly used in EC2

Free naming. Common names: Name, Environment, Team

For:

- Resource Grouping

- Automation

- Cost Allocation

### Resource Groups

You can create, view or manage logical Group resources based on Tags:

- Group apps together

- Group different layers of an app stack

- prod and dev environments

Works on regional level.

Activity:

1. EC2 > instances > Manage tags, create/remove tags

2. in AWS Resource Groups > Create > Tag based

2. In SSM > Resource groups > Create > Assign Group Criteria (eg. Environment = dev)

### SSM: Documents

Can be in JSON or yaml

- define parameters

- define actions

- Many documents already exists in AWS

Documents can be used to:

- Run Commands

- Manage State

- Patch Manager

- Automation

- Parameter store

Activity:

1. SSM > Shared Resources > Documents

2. Applying SSM Documents:

    - use `Run Command`

        - Run a document (a collection of scripts) or just a single command

    - can run comman across multiple instances (using resource groups)

    - Rate Control / Error Control

    - Integrated with IAM and CloudTrail

    - NO need for SSH

    - Command Output can be:
    
        - shown in the COnsole, sent to S# Bucket or CLoudWatch logs

        - Send notifications to SNS about command status (In progress, success, failed)

        - Can be invoked using EventBridge

3. Install HTTP server onto all instances in the group

    - Ensure your instances have appropriate Inbount Rules
    
    - In SSM > Create Document (Command or Session) > COmmand Document

    ```yml
    ---
    schemaVersion: '2.2'
    description: Sample YAML template to install Apache
    parameters: 
    Message:
        type: "String"
        description: "Welcome Message"
        default: "Hello World"
    mainSteps:
    - action: aws:runShellScript
    name: configureApache
    inputs:
        runCommand:
        - 'sudo yum update -y'
        - 'sudo yum install -y httpd'
        - 'sudo systemctl start httpd'
        - 'sudo systemctl enable httpd'
        - 'echo "{{Message}} from $(hostname -f)" > /var/www/html/index.html'
    ```

    - In SSM > RUn Command > Run the Document you created

        - Choose a resource group or individual instance

### SSM: Automation

Simplifies common maintenance and deployments tasks of EC2 instances and other AWS Resources.

Eg: Restart instances, create an AMI, EBS Snapshot

Uses *Automation Runbook*

- SSM Documents of type Automation

- defines actions performed on your EC2 instances or AWS resources

- use pre-defined runbooks created by AWS or create custom runbooks

- Can be triggered:

    - manually using AWS Console, CLI or SDK

    - by Amazon EventBridge

    - On a schedule using Maintenance Windows

    - BY AWS Config for rules remediations

Activity:

    - Go to Systems Manager (SSM) > Automation > Execute Automation

    - Choose document > categories (Instance Management) > `AWS-RestartEC2Instance`

    - or filter Document Type: Automation

    - Execution method:

        - Simple

        - Rate Control

            - Execute safely on multiple targets on multiple concurrency and error thresholds.

        - Multi Account and Region

        - Manual

Use Cases:

1. Reduce costs by automatically start and stop EC2 instances and RDS Db Instances

    - at 9am: use EventBridge to Schedule SSM Automation to start instances (`AWS-StartEC2Instance`, `AWS-StartRdsInstace`)

    - at 5pm: use EventBridge to Scedule Stop (`AWS-StopEC2Instance`, `AWS-StopRdsInstance`)

2. Reduce Costs by automatically downsize EC2 instances and RDS Db Instances (`AWS-ResizeInstance`)

3. Build a golden AMI

    - EventBridge invoke Run Automation `AWS-CreateImage` every week

    - When the AMI is available, it triggers EventBridge to invoke Lambda Function with that ami-id

    - store that ami-id in Parameter Store

4. Integration with AWS Config (Config Rule)

    - Reenable versioning if non compliant `AWS-ConfigureS3BucketVersioning`

### SSM: Parameter Store

Secure storage for configuration and secrets.

OPtional Seamless Encryption using KMS.

Serverless, scalable, durable, easy SDK.

Version tracking of configurations / secrets

Security through IAM (for eg. EC2's execution Role).

Integrations:

    - Notification with Amazon EventBridge

    - Integration with CloudFormation

1. Store Hierarchy

    - eg:

        - `/my-department`

            - `/my-app`

                - `dev`

                    - `db-url`

                    - `db-password`

                - `prod`
                    
                    - `db-url`

                    - `db-password`
            - other-app
        - other department

    - Access Secrets of Secrets Manager: `aws/referece/secretsmanager/secret_ID_in_Secrets_Manager`


2. Tiers: Standard vs Advances

    - main difference:

        - no. of parameters allowed

        - max size of parameter value

        - Policies only avaliable for Advanced

            - Parameters Policies:

                - Allow to assign TTL to a parameter (expiration date) to force updating or deleting sensitive data such as passworlds

                - can assign multiple policies at a time

                - eg. policies:

                    - Expiration - to delete parameter

                    ```json
                    {
                        "Type": "Expiration",
                        "Version": "1.0",
                        "Attributes": {
                            "Timestamp": "2020-12..."
                        }                    
                    }
                    ```

                    - ExpirationNotification - EventBridge

                    ```json
                    {
                        "Type": "ExpirationNotification",
                        "Version": "1.0",
                        "Attributes": {
                            "Before": "15",
                            "Unit": "Days"
                        }                    
                    }
                    ```

                    - NoChangeNotification - EventBridge

                    ```json
                    {
                        "Type": "NoChangeNotification",
                        "Version": "1.0",
                        "Attributes": {
                            "After": "15",
                            "Unit": "Days"
                        }                    
                    }
                    ```

3. Activity:

- Go to SSM > Parameter Store > Create Parameter

- Types: String, StirngList, SecureString (encrypted)

- Specify Hierarchies in the `name`

- Select Standard or Advanced

- Using CLI:

```bash
aws ssm get-parameters --names /my-app/dev/db-url /my-app/dev/db-password

# automatically decrypt securestring value
aws ssm get-parameters --names /my-app/dev/db-url /my-app/dev/db-password --with-decryption

# get all parameters on a specific path
aws ssm get-parameters-by-path --path /my-app/dev --recursive
```

### SSM Patch Manager

Automates the process of patching managed instances

Eg: OS Updates, applications updates, security updates...

Supports both EC2 instances and no-premise servers

Supports Linux, MacOS, Windows

Patch on-demand or ona schedule using Maintenance Windows

Scan instances and generate *patch compliance report*. This report can be sent to S3 for further action.

Components:

1. Patch Baseline

    - Defines which patches should and should not be installed on your instances

    - ability to create custom Patch Baselines (specify approved/rejected patches)

    - patches can be auto-approved within dats of their release

    - By default, install only critical patches and patches related to security

    - If a Patch Baseline is not attached to any Patch Groups, it is attached to the Default Patch Group

    - 2 types:

        - Pre Defined Patch Baseline

            - Managed by AWS for different OS - cant be modified

            - `AWS-RunPatchBaseline` (SSM Document) - apply both os and application patches (Linux, MacOS, Windows Server)

        - Custom Patch Baseline

            - Create your own Patch Baseline and choose which patches to auto-approve

            - OS, allowed patches, rejected patches..

            - Ability to Specify cusatom and alternative patch repositories

2. Patch Groups

    - Associate a set of instances with a specific Patch Baseline

    - eg: create Patch Groups for different environemtns (dev,test, prod)

    - Instances should be defined with the tag key *Patch Group*

    - an instance can only be in ONE Patch Group

    - Patch Group can be registered with only ONE Patch Baseline

(AWS Console or SDK or Maintenance Window) --> Initiates `run Document (AWS-RunBatchBaseline)` -> Applies patch installation, based on Patch Manager configurations

3. Maintenance Windows

    - Define a schedule for when to perform actions on your instances

    - eg: OS Patching, updating drivers, installing software

    - Contains:

        - Schedule

        - Duration

        - Set of registered instances

        - Set of registered tasks

    - Activity

        - Go to Patch Manager > Patch Now or `View Predefined Baselines`

        - In SSM > Maintenance Window > Configure Schedule

### SSM Session Manager

Allows you to start a secure shell on your EC2 and on premise servers.

Access through AWS Console, AWS CLI, Session Manager SDK.

Important: Does not need SSH Access, bastion hsots, or SSH Keys

EC2 instance must have SSM Agent, Users connect to the instance via SSM Session Manager.

Supports Linux, MacOS, Windows

Log Connections to your instances and executed commands

Session log data can be sent to S3 or CloudWatch Logs.

CloudTrail can intercept *StartSession* events.

1. Reqquirements:

    - IAM Permissions:

        - Control which users/group can access Session Manager and which instances

        - Use tags to restrict access to only specific EC2 instances

        - Access to SSM + write to S3 + write to CloudWatch

    - You can also (optional) restrict commands a user can run in a session

    - instance DOES NOT need Inbound Rules configured

Benefits:

1. Improved security

2. Logging capabilities which can be sent to S3 or CLoudWatch Logs

Activity:

1. SSM > Session Manager > Start Session > Choose instance


### SSM Default Host Management (DHMC)

When enabled, it automatically configure your EC2 instances as managed instances without the use of EC2 Instance Profile.

*Instance Identity Role* - a type of IAM Role with NO permissions beyond identifying the EC2 instance to AWS Services (eg. Systems Manager)

    - Passes an IAM Role `AWSSystemsManagerDefault`, `EC2InstanceManagementRole`, enables the instance to be controlled by Systems Manager.

Requirements:

1. EC2 instances must have IMDSv2 enabled

2. Must have SSM Agent Installed (DOES NOT support IMDSv1)

Automatically enables Session Manager, Patch Manager, and Inventory

Automatically keeps the SSM Agent up to date

Must be enabled per AWS Region

Activity:

    - AWS SSM > Fleet Manager > Click `Configure Default Host Management`

    - EC2 Console > Instances > launch Instance > choose Amazon Linux

        - In Metadata > Enabled > Version V2 > Launch

        - Upgrade the SSM Agent on the instance if required

        ```bash
        sudo systemctl stop amazon-ssm-agent
        
        # install latest ssm agent
        sudo yum install -y https:....
        ```

    - In AWS SSM > the instance should appear in the Fleet Manager

### SSM Hybrid Environments

You can setup Systems Manger to manage on-premises servers, IoT devices, edge devices, and VMs (eg. VMs in outher cloud providers)

In Systems Manager Console, EC2 instances use the prefix "i-" and hybrid managed nodes use the prefix "mi-".

Steps:

1. Administrator create a Hybrid Activation on Systems Manager

2. SSM with return Activation Code & ID

3. Install SSM Agent on the target server

4. Register with SSM using Activation Code and ID obtained in step 2.

Or you can automate this using architecture:

[On Premise Server] -->HTTP GET Request--> [AWS API GW] --> sends Payload to [AWS Lambda] --> Create Hybrid Activation on [SSM]

[SSM] returns Activation Code and ID to [AWS Lambda] --> forwards to [AWS API GW] --> returns to [On Premise Server] --> Registers itself

Activity:

1. SSM > Node Management > Hybrid Activation > Create Activation

2. On EC2 > launch New Instance - no IAM Role

3. Install SSM Agent

```bash
# if you need to install the SSM Agent manually:
sudo yum install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm

# check status ubuntu
sudo systemctl status snap.amazon-ssm-agent.amazon-ssm-agent.service
sudo snap stop amazon-ssm-agent
```

4. Register the instance

```bash
# edit the code, id and region in the command below
sudo /snap/amazon-ssm-agent/current/amazon-ssm-agent -register -code "activation-code" -id "activation-id" -region "region" 
sudo snap start amazon-ssm-agent
```

5. Your launched instance now should appear in the Fleet Manager

### SSM IOT GreenGrass

Manage IoT Greengrass Core devices using SSM

Install SSM Agent:
    
    - can be installed manually or 
    
    - deployed as a Greengrass Component (pre-built software module tha tyou deploy directly to Greengrass Code devices)

Once installed, you must add permissions to the *Token Exchange Role* (IAM Role for the IoT core device) to communicate with SSM

Supports All SSM Capabilities: Patch Manager, Session Manger, Run Command, etc

Example Use case: easily update an maintain OS and software updates across a fleet of Greengrass Core devices

### SSM Compliance

Scan your fleet of managed nodes for patch compliance and configuration inconsistencies

Displays current data about:

- Patches in Patch Manager

- Association in State Manager

Can sync data to an S3 bucket using Resource Data Sync, and you can analyze using Athena and QuickSight

Can collect and aggregate data from multiple accounts and regions.

Can send compliance data to Security Hub

### SSM OpsCenter

Allows you to view, investigate and remediate issues in one place (no need to navigate across different AWS services)

Security Issues (from Security Hub), performance issues (DynamoDB throttle), failures (ASG failed launch instance)...

Reduce meantime to resolve issues

OpsItems

    - Operational issue or interruption that needs investiogation and remediation

    - Event, resource, AWS Config changes, CloudTrail logs, EventBridge...

    - Provides recommended Runbooks to resolve the issue

Supports both EC2 isntances and on premise nodes

Ops center can then trigger SNS.

1. Example:

    - EventBridge invoke a Lambda function daily --> List EBS volumes and search for aged EBS Volumes (eg. over 45 days old) --> if there is, lambda function create OpsItem in OpsCenter

    - OpsCenter then runs Document `AWS-CreateSnapshot` or `AWS-DeleteSnapshot` as SSM Automation.

### SSM Session Manager: VPC Endpoint

If you want to access EC2 instances that are in *private subnet* without Internet access.

Use VPC Endpoint.

1. SSM Service must have VPC Interface Endpoint. Allow inbound 443 in the security group.

2. Another VPC Endpoint that is required is on the SSM Session Manager. Allow inbound 443 in the security group.

3. If you are using KMS Encrytion, we must also create VPC Interface Endpoint for KMS. 

4. If you use CloudWatch Logs, we must also create VPC Interface Endpoint for that service.

5. If the logs are sent to S3, we can use the VPC Gateway Endpoint for Amazon S3. Note: for  S3 Gateway Endpoint, you need to update route tables.


## 2.9: AWS OpsWorks

3 Main Services:

1. OpsWorks Stacks <-- only this in in Exam

2. OpsWorks for Chef Automate

3. OpsWorks for Puppet Enterprise

A Stack: A Set of Layers, instances and related AWS resources whose configs you want to manage together.

OpsWorks Stacks - Use Chef cookbooks to deploy your applications.

Stack has many layers.

- manage infra, config, and applications in one place.

- Basic Layers:

    1. Load Balancer - AWS Specific (shared layer)

    2. Application Server Instances -  many instances -not AWS specific - where the app sits

        - You can provision a *Chef Cookbooks* (optional)

        - It could be in git, HTTP Archive or S3 Archive

    3. Amazon RDS Instance - AWS Specific (shared layer)

    - You can add layer, either a:

        1. OpsWorks Layer

        2. ECS Layer

        3. RDS Layer

- Stack Settings

- Layers

    - Auto Healing - if an instance fails, it will be automatically healed. ie. The instance in stopped, and verifies that it has stopped. The, the instance with be started.

    - Every instance has an AWS OpsWorks Agent that communicates regularly with the service to minitor health. If an agent does not communicate with the service for more than approximately 5 minutes, AWS OpsWorks Stacks considers the instance to have failed.

- Instances > Click on the instance

Instances are managed by OpsWorks. Types of instances:

1. Time-based

    - OpsWorks automatically starts and stops instances based on a specified schedule.

    - Define Schedule - eg. Between 9am to 10pm

2. Load-based

    - OpsWorks automatically starts and stops instance in response to CPU, memory, and application load

In OpsWorks, instances are NOT auto-scaled. YOu need to know before hand how many servers you need.

### OpsWorks Stacks Lifecycle Events

Each layer has a set of FIVE lifecycle events, each of which has an associated set of recipes that are specifc to the layer.

Go to Layers > Select a Layer > Click the *Recipes* tab.

If you click Edit, you are able to bind each of the lifecycle events to a *Cookbook*

There are 5 lifecycle events:

1. Setup

    - when a started instance has finished booting

    - happens on individual instance

2. Configure (!important)

    - This event occurs on *ALL* of the stack's instances when one of the folowing occurs:

        - an instance enters or leaves and online state

        - You *associate an Elastic IP address with an instance* or disassociate one from an instance

        - You attach an ELB to a layer, or detach one from a layer

3. Deploy

    - occurs when you run a *Deploy* command.

    - Note: Setup also includes Deploy

    - happens on individual instance

4. Undeploy

    - occurs when you delete an App or run *Undeploy* command

    - happens on individual instance

5. Shutdown

    - occurs when we shut an instance down, but before associated Amazon EC2 instance is actually terminated.

    - OpsWorks Stacks runs recipes to perform *cleanup* tasks such as shutting down services.

    - happens on individual instance

To delete in instance, you must stop it first.


### OpsWorks and CloudWatch Events Integration

Refer to Auto Healing above.

Go to CloudWatch > create a Rule > service name: OpsWorks > event type: OpsWorks Instance State Change.

Specify targets: SNS Topic to send email notif.


