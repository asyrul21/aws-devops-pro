# Domain 1: SDLC Automation & CI/CD

Main services:

1. CodeCommit

2. CodePipeline - Orchestrate CI/CD

3. CodeBuild 

- Build and Test

- Similar to Jenkins CI,

4. CodeDeploy

- Deploy to EC2, AWS Lambda, ECS, etc.

5. CodeStar

6. CodeArtifact

7. CodeGuru

## 1.1 Code Commit

A Git repository.

1. Version control, rollback

2. Stores Code

3. Similar to GitHub, Bitbucket, GitLab

- default branch: `main`

### Specialties:

1. Private repositories

2. No size limit

3. Fully Managed, highly available

4. Security

- Authentication

    - ssh keys

    - HTTPS

- Authorization

    - IAM Policies

- Encryption using KMS

- Cross-account Access

5. Integrated with Jenkins, AWS CodeBuild, etc

### Features

1. Commit Visualizer

2. Compare Commits

3. Settings

    - Notificaition Rules

        - Targets

            - SNS Topic / Slack Chatbot

4. Triggers

    - Event-based triggers

5. Tags

### Activities

1. Create Repo

2. Upload File and Commit

3. Create Branch

4. Settings > Notification Rules


### Use CLI

1. Setup IAM Security Credentials: You can use Git SSH or HTTPS

- Go to AWS > IAM > Your User > Security Credentials tab

- Under `SSH public keys for AWS CodeCommit`, upload your SSH.pub key

- ED25519 does not work, need to use `rsa` key

- for HTTPS, generate username/password under `HTTPS Git Credentials for AWS CodeCommit`, then download credentials

### Advanced

1. Monitor CodeCommit events with EventBridge

2. Migrating to CodeCommit from another Client (Github/Bitbucket/etc)

    1. Create a repository on CodeCommit

    2. Git clone to local from  another client

    3. Push from local to repo created in (1)

3. Cross Region Replication - lower latency pulls

    1. When a developer push to make changes to a branch, code commit emits an event to Event Bridge

    2. From EventBridge, we can invoke ECS Task to git clone the repo, and push it to another replica:

    `git remote set-url --push origin [new replica]`

4. Branch Security

    1. Use IAM Policies to control which branches are allowed to specific developers

    2. By default, a user who has push permissions to a CodeCommit repo can contribute to any branch

    ```json
    {
        "Statement": [
            {
                "Effect": "Deny",
                "Action": [
                    "codecommit:GitPush",
                    "codecommit:DeleteBranch"
                    ...
                ]
            }
        ]
    }
    ```

    3. Attach the Policies to junior devs users.

5. Pull Request Approval Rules

    1. Specify for eg. mininum number of required approvers to approve before PRs can be merged.

    2. Can use Approval Rule Templates


## 1.2 Code Pipeline

A Visual Workflow to orchestrate your CICD.

- Source - CodeCommit, ECR, S3, Bitbucket, Github

- Build - CodeBuild, Jenkins, CloudBees, TeamCity

- Test - CodeBuild, AWS Device Farm, 3rd Party tools

- Deploy - CodeDeploy, Elastic Beanstalk, CloudFormation, ECS, S3..

Consists of *stages*

### Artifacts

Whatever that is created by any of the pipeline stages.

Each stage consume and output artifacts to S3.

### Troubleshooting

1. Can use CloudWatch Events (Amazon EventBridge)

2. Examples:

    - create events for failed pipelines

    - create events for cancelled pipelines

    - get notified/emails etc

3. To Audit AWS API Calls - use AWS CloudTrail

### Activities

1. On ElasticBeanstalk, create environtments - one for dev, and another for prod

2. Create New Pipeline, integrate with Elastic Beanstalk

3. Add Stage to pipeline

### Event-Driven Implementation

1. Events vs Webhooks vs Polling

- Events 

    - preferred way in CodePipeline - default

    - eg. commit is pushed to CodeCommit, an event is emitted to EventBridge, which triggers CodePipeline stage

    - for Github, use CodeStar Source Connection (Github App) -> triggers CodePipeline

- Webhook

    - CodePipeline exposes an Http Endpoint, which can be triggered by a script to do something

- Polling

    - Regular checks by CodePipeline on Github

2. Action Type Constrainst for Artifacts

3. CodePipeline Manual Approval (Exam Question)

    - Important: Owner must be *AWS* and Action is *Manual*

    - Approver must have 2 permissions (IAM User Policy) :

        - codePipeline:GetPipeline*

        - codePipeline:PutApprovalResult

### Cloud Formation Integration

- CloudFormation can be used as a Target for CodePipeline

- *CloudFormation Deploy Action* can be used to deploy AWS resources eg. Lambda function

- eg. Deploy Lambda functions using CDK or SAM (alternative to CodeDeploy)

- CodeCommit -> Create *Change Set* in CloudFormation -> Manual Approval -> Execute Change Set

- CloudFormation *StackSets* to deloy across multiple AWS accounts and regions

- CodeCommit -> CodeBuild (create templates for CloudFormation for each region) -> CloudFormation (consumes the templates as input) -> Deploy a Lambda Function

- Action Modes

    - Create or Replace a Change Set, Execute a Change Set

- Template Parameter Overrides
    
    - Specify a JSON object to override parameter values

    - Retrieves the parameter value from *CodePipeline Input Artifact*

    - All parameter names but be present in the template

    - Static - use template configuration file (recommended)

    - Dynamic - use parameter overrides

### Advanced

1. Best Practices

    - Common Patterns:

        - One CodePipeline -> One Code Deploy -> Parallel Deploy to multiple *Deployment Groups*

        - Parallel Actions using in a Stage using *RunOrder*

        - Deploy to Pre-Prod before deploying to Prod

2. Use EventBridge to detect changes in execution states, for debugging and/or detect failures

    - we can detect and *intercept* those changes (eg. failures at specific stage) and invoke a Lambda to diagnode code or trigger SNS to notify user.

3. If you want to call an API from CodePipeline, best way is to use *Invoke Action* to invoke Lambda function within a Pipeline

4. Use *StepFunctions* to start State Machine within a Pipeline

5. Multi Region via CloudFormation

    - Actions in pipeline can be in different regions. eg. deploy a Lambda function through CloudFormation into multiple regions

    - S3 Artifact stores must be defined in each region where you have Actions. CodePipeline must have read and write access into every artifact bucket

    - CodePipeline handles the copying of input artifacts from one AWS Region to the other regions when performing cross-region actions.


## 1.3 Code Build

Builds code / source from CodeCommit / Github / Bitbucket.

Build instructions:

    - (exam question) name must be *buildspec.yml*

    - create or insert it manually (in code repo)

Ouput logs can be stored in: S3 and/or CloudWatch Logs.

Use CloudWatch Metrics to monitor build statistics.

Use CloudWatch Alarms to notify if you need threshold for failures.

*Build Project* can be defined within CodePipeline or (either) CodeBuild.

### Environments

- Can use pre-built images for supported languages

- to extend, use Docker          

### Basic Flow

Code Commit (contains code + buildspec.yml) -> Code Build (pulls a docker image based on environment and runs the buildspec.yml) ->

- You can cache some files to S3 Bucket (optional)

- Store logs in Amazon S3 or CloudWatch Logs

- upload artifacts to S3 Bucket

### buildspec.yml

1. Must be at root of project

```yaml
version: 0.2

#
# env: environment variables
#
env:
    # plaintext variables
    variables:
        JAVA_HOME: "/usr/lib/jvm/..."
    # variables stored in SSM Parameter Store
    parameter-store:
        LOGIN_PASSWORD: /path/in/SSM/param/store
    # variables stored in AWS Secrets Manager
        MY_SECRET: /path/to/secret/manager

#
# phases - build steps / jobs
#
phases:
    # install dependencies
    install:
        commands:
            - echo "Entered the install phase..."
            - apt-get update -y
            - apt-get install -y maven
    # final commands to execute before build
    pre_build:
        commands:
            - echo "Entered the pre_build phase..."
            - docker login -u User -p $LOGIN_PASSWORD
    # build commands
    build:
        commands:
            - echo "Entered the build phase..."
            - mvn install
    # post build
    post_build:
        commands:
            - echo "Entered the post_build phase..."
            - echo "Build completed on `date`"

#
# artifacts: what to upload to S3
#
artifacts:
    files:
        - target/my-artifact.jar

#
# cache: which files to cache to S3 for future build speedup
#
cache:
    paths:
      - "/root/.m2/**/*"    
```

### Running Local Build

1. Install Docker, run CodeBuild locally

2. Use CodeBuild Agent

### CodeBuild and VPCs

- By default, CodeBuild is run outside of VPC. Hence, it cannot access resources in a VPC

- Specify VPC Configuration:

    - VPC ID

    - Subnet IDs

    - Security Groups IDs

### Activities

1. Create a Build Project

```yml
version: 0.2

phases:
    install:
        runtime-versions:
            nodejs: 10
        commands:
            - echo "Entered the install phase..."
    pre_build:
        commands:
            - echo "Entered the pre_build phase..."
    build:
        commands:
            - echo "Entered the build phase..."
            - echo "we will run some tests"
            - grep -Fq "Congratulations" index.html
    post_build:
        commands:
            - echo "Entered the post_build phase..."
```

2. Integrate Build Project into CodePipeline

    - Pipeline > Edit > Add a Stage (BuildAndTest)

    - Add Action Group > Action Provider (CodeBuild) > Select Region > > Input Artifact (Source Artifact) > Select Build Project name > Output Artifact (enter name)

### Advanced

1. Environment Variables

    - Default variables: AWS_DEFAULT_REGION, CODEBUILD_BUILD_ARN, CODEBUILD_BUILD_ID, CODEBUILD_BUILD_IMAGE...

    - Custom variables

        - Static - defined at build time (overrride using start-build API call)

        - Dynamic - using SSM Parameter Store or AWS Secrets Manager

2. Security

    - CodeBuild Service Role - allows CodeBuild to access AWS resources on your behalf

    - in-transit and at-rest data encryption

    - Build output artifact encryption

3. Build Badges

    - dynamically generated, shows status of latest build

    - can be accessed through public URL for your CodeBuild project

    - Supported for CodeCommit, Github and Bitbucket

    - Badges are available at BRANCH level

4. Triggers - how to trigger CodeBuild

    - when CodeCommit changes (eg commit pushed), event is emitted to EventBridge, can trigger CodeBuild

    - when CodeCommit changes (eg commit pushed), event is emitted to EventBridge, can trigger Lambda Function, which invokes CodeBuild

    - from Github -> event Webhook -> trigger CodeBuild

5. Using CodeBUild to Validate PR:

    PR from develop to main created on CodeCommit repo -> PR event emitted to EventBridge -> 
    
        - (1) trigger Lambda function -> update PR with comment (test build begin)

        - (2) trigger CodeBuild -> success or failure event emitted to EventBridge -> invoke a Lambda function with updates the PR with comment (build outcome)

6. CodeBuild Test Reports UI

    - contains details about tests that are run during build

    - create your test cases with any test framework - JUnit XML, NUnit XML, etc

    - Cucumber JSON, Visual Studio TRX

    - Create a test report and add a *Report Group* name in buildspec.yml

## 1.4 Code Deploy

Deployment service with automation.

Delpoyments can be made to: EC2 Instances, On Premise servers, Lambda functions, ECS Services.

Automated Rollback capability in case of failed deployments, or trigger CloudWatch Alarm.

Gradual deployment control.

Use file: `appspec.yml` to define how deployment happens.

1. EC2 or On Premise

    - allows 2 types of deployments: *in-place* and *blue/green*

    - Must run *CodeDeploy Agent* on the target instances

    - Define deployment speed:

        - AllAtOnce: Most Downtime

        - HalfAtATime: reduce capacity by 50%

        - OneAtATime: slowest, lowest availablility impact

    - Examples:

        - In Place, Half at a time: 
        
        v1,v1,v1,v1 -> x.x.v1.v1 -> v2.v2.v1.v1 -> v2.v2.x.x -> v2.v2.v2.v2

        - Blue Green: Create new Auto-scaling group (ASG), as many ec2 instance in original group, with v2. Then, ALB points to new group.

    - CodeDeploy Agent must be installed and running on the EC2 instances as a pre-requisite

        - can be installed and updated manually or automatically using System Manager

        - EC2 instances must have sufficient permissions to access Amazon S3 to get deployment bundles. S3 is used to store the Application Revision. Policy:

        ```json
        {
            "statement": [
                {
                    "Action": [
                        "s3:Get*",
                        "s3:List*"
                    ],
                    "Effect": "Allow",
                    "Resource": "*"
                }
            ]
        }
        ```

2. Lambda Platform

- can help automate *Traffic Shift* for Lambda aliases

    - CodeDeploy will make *X* vary from v1 to v2 of `PROD Alias` starting from 0% to 100%

    - Strategies:

        - Linear: grow traffic every N minutes until 100%

        - Canary: try X percent, then 100%, over Y Minutes

            - LambdaCanary10Percent5Minutes

            - LambdaCanary10Percent30Minutes

        - All At Once: immediate

- feature is integrated within SAM framework

3. ECS

- can help automate automate the deployment of a new ECS Task Definition

- Only *Blue/Green* Deployments: CodeDeploy creates a copy TargetGroup (green), based on original (Blue), with same number if instances. Then CodeDeploy adjusts the ALB to redirect requests from Blue target group to Green target group.

- Strategies:

    - Linear: grow traffic every N Minutes

        - ECSLinear10PercentEvery3Minutes

        - ECSLinear10PercentEvery10Minutes

    - Canary: Try X percent, then 100%

        - ECSCanary10Percent5Minutes

        - ECSCanary10Percent30Minutes

    - AllAtOnce: immediate

### EC2 Deep Dive

1. In Place deployment:

    - Must use EC2 Tags or part of ASG to identify instances you want to deploy to

    - In-Place Deployment Hooks

        - Hooks are one or more scripts to be run by CodeDeploy on each EC2 instance

            - with ALB: [BeforeBlockTraffic, AfterBlackTraffic], ApplicationStop, BeforeInstall, AfterInstall, ApplicationStart, ValidateService, [BeforeAllowTraffic, AfterAllowTraffic]

            - no ALB: ApplicationStop, BeforeInstall, AfterInstall, ApplicationStart, ValidateService

        - Hooks are defined as script paths in *appspec.yml*

        - *CodeDeploy Agent* execute the hook scripts, and inject env variables. eg DEPLOYMENT_GROUP_NAME

2. Blue Green Deployment:

    - Manual Mode: provision EC2 instances for BLue and Green and identify by Tags

        - instances are grouped by Tags { name: projectName, value: MyApp_v1 }, ALB gradually shifts from instances from MyApp_v1 tag to MyApp_v2 tag.

    - Auto mode: instances  are grouped in a Auto Scaling Group (ASG).

    - *IMPORTANT* for Blue Green Deployments, YOU MUST HAVE ALB

    - Instance termination

        - BlueInstanceTerminationOption

        - Action Terminate - Specify Wait Time, default 1 hour, max 2 days

        - Action Keep Alive - instances are kept running but deregistered from ELB and deployment group

    - Hooks - Same as In Place deploytment, but in specific order:

        - Ran on Green (v2) instances (they have no traffic initially): ApplicationStop, BeforeInstall, AfterInstall, ApplicationStart, ValidateService, [BeforeAllowTraffic, AfterAllowTraffic]

        - Ran on Blue (v1) instances: [BeforeBlockTraffic, AfterBlackTraffic]

        - Hooks are defined as script paths in *appspec.yml*

    - Deployment confiurations - specify number of instances that must remain available at any time during the deployment

        - CodeDeployDefault.AllAtOnce

        - CodeDeployDefault.HalfAtATime

        - CodeDeployDefault.OneAtATime

        - create your own

    - CodeDeploy publishes Deployment / EC2 instance events to SNS Topic, which can send email to notify

### ECS Deep Dive

Automatically handles a deployment of a new ECS Task Definition to an ECS Service.

Only supports Blue/Green - service MUST be connected toa Load Balancer.

ECS Task Definition and new Container images must be already created.

CodeDeploy Agent NOT required

1. Example:

    1. Developer pushes new image to ECR

    2. Create ECS Task Defintion Revision to reference (1)

    3. Create `appspec.yml` that references the new Task Definition/Revision in (2) and load balancer information (LoadBalancerInfo)

    4. The `appspec.yml` must be placed in S3 Bucket.

    5. CodeDeploy will reference the `appspec.yml` in (4)

    5. ECS deploy to ECS cluster, deleting old ECS Task

2. Flow:

    1. Developer pushes code

      |
      V

    -----------------------------
    Code Pipeline

    2. Code Commit

    3. Code Build

        - Build Docker Image

        - Create New ECS Task Defintion Revision

        - Create `appspec.yml` which reference the built image. This file will be placed in S3 bucket

    4. Code Deploy

        - reference as input - the `appspec.yml`
    -------------------------------
      |
      V
    5. Deploy and Shift traffic to ECS Cluster

3. Deloyment strategies to shift traffic to new Task Set

    - Canary

        - ECSCanary10PercentEvery5Minutes

        - ECSCanary10PercentEvery15Minute

    - Linear

        - ECSLinear10PercentEvery1Minute

        - ECSLinear10PercentEvery3Minute

    - All At Once

        - ECSAllAtOnce

4. Hooks - MUST BE LAMBDA FUNCTIONS

    - before install, after install, AfterAllowTestTraffic, BeforeAllowTraffic, AfterAllowTraffic

    ```yml
    # appspec.yml

    Hooks:
      - AfterInstall: "ValidateAfterInstall" # <- this is name of Lambda Function to invoke
    ```

### Lambda Deployment Deep Dive

Only Blue-Green Deployment supported.

Scenario: we want to create a Lambda Function, and want to deploy it onto an Alias

Specify version info of the new Lambda Function in the `appspec.yml`. Placed in S3 bucket.

In `appspec.yml`, invoke CodeDeploy, which updates the Lamda Alias to reference the new Lambda version.

Traffic will be shifted automatically.

```yml
# appspec.yml

Resources:
    - myLambdaFunction:
      Type: AWS::Lambda::Function
      Properties:
        Name: nameOfLambdaFunc
        Alias: LambdaAliasName
        CurrentVersion: 1
        TargetVersion: 2
```

1. Flow

    1. Push Code

      |
      V

    ------------------------
    CodePipeline

    2. CodeCommit

    3. CodeBuild

        - Build the Lambda Code (Version 2)

        - Create `appspec.yml`, place it in S3

    4. CodeDeploy

        - takes the `appspec.yml` as input

        - Deploy on Alias and shifts traffic

    -----------------------

      |
      V
    5. Lamda Alias

2. Deployment Strategies for shifting traffic to new version:

    - Canary

        - LambdaCanary10PercentEvery(5,10,15,30)Minute

    - Linear

        - LambdaLinear10PercentEvery1Minute

        - LambdaLinear10PercentEvery(2,3,10)Minute
    
    - AllAtOnce

        - LamdaAllAtOnce

3. Hooks - As Lambda Functions - executed once per deployment

    - BeforeAllowTraffic - can be used to do health check

    - AfterAllowTraffic - can be used to do health check


### Redeploy and Rollbacks

Rollback = redeploy a previously deployed revision of your application.

Types:

    1. Automatically- rollback when a deployment fails or when a CloudWatch Alarm thresholds are met

    2. Manually

Can disable rollback

If rollback happens, CodeDepoy redeploys the last known *Good Revision* as a *NEW* deployment. NOT a Restored version

### Troubleshooting

1. DeploymentError: InvalidSignatureException - [time] is now earlier than [time]

    - CodeDeploy requires accurate time references

    - if date and time on your EC2 instances are not set correctly, they might not match the signature date of your deployment request

2. Deployment or all Lifecycle Events skipped

    - errors:  too many individual instances failed deployment

    - error: too few healthy instances are available for deployment

    - error: Some instances in your deployment group are experiencing problems

    - `CodeDeploy Agent` might not be installed, running or cant be reached by CodeDeploy

    - Code Deploy Service Role or IAM instance profile might not have the required permissions

    - You are using an HTTP Proxy, configure CodeDeploy Agent with `:proxy_uri:` parameter

    - Date and Time match between CodeDeploy and Agent

3. ASG - If CodeDeploy deployment to ASG is underway and a *Scale-Out* event occurs, the new instances will be updated with the application revision that was most recetly deployed (not the application revision that is currectly being deployed)

    - ASG will have EC2 instances hosting different versions of the app

    - by default, CodeDeploy automatically starts a *Follow-On Deployment* to update any outdated EC2 instances

3. AllowTraffic is failing without error reported in Deployment Logs

    - due to incorrectly configured health checks in ELB

    - review and correct any errors in ELB health checks configurations



## 1.4 CodeArtifact

Reusable software package / library repository.

Secure, scalable, cost effective Artifact Management.

Works with common dependency management tools: Maven, Gradle, NPM, yarn, pip, NuGet etc

Developers and CodeBuild can retrieve packages directly from CodeArtifact

--------------------------------------------
| VPC
|   -----------------------------------
|   |   AWS CodeArtifact 
|   |   ---------------------------------
|   |   | Domain:
|   |   |   - repository A
|   |   |   - repository B
|   |   |  ___________________________
|   |   | | Shared repository storage |

Code Artifact as Proxy:

Repo -> CodeArtifact -> NPM

 - Network security

 - Cached into CodeArtifact - even if NPM library is removed, it still exists in CodeArtifact

 - IT leader can publish / approve packages

 ### EventBridge Integration

 1. When a Package is created, modified, or deleted, CodeArtifact emits events to EventBridge

 2. This events can:

    - invoke Lambda Function

    - activate Step Functions State Machine

    - send message to SNS

    - send message to SQS

    - Start a CodePipeline Pipeline

        - CodeCommit

        - CodeBuild

        - CodeDeploy

3. Any artifact repo acan be accessed by any user account with IAM Policy

    - You can also use *Resource Policy* to authorize another account to access CodeArtifact

    - A given principal can either READ ALL packages in a repo or NONE of them

```json
// resource policy
{
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "codeartifact:DescribePackageVersion",
                "codeartifact:DescribeRepository",
                //...
            ],
            "Principal": {
                "AWS": [
                    "arn:aws:iam:123123123:root",
                    "arn:aws:iam:123123123:user/bob",
                ]
            },
            "Resource": "*"
        }
    ]
}
```

### Upstream Repositories

1. A CodeArtifact repo can have other CodeArtifact repo as Upstream Repo

2. Allow a package manager client to access the packages that are contained in more than one repository using a single repository endpoint

3. Up to 10 upsteam repo per Repo

4. Can only be ONE External Repository Connected. eg. NPM

    - External Connection - CodeArtifact repo connects to external repo - NPM, Maven, etc

    - Allows us to fetch and cache packages that not already present in CodeArtifact Repo

### Retention

1. If a requested package version is found in an Upstream Repo, a reference to it is retained and ALWAYS available from Downstream Repository

2. The retained package version is not affected by changes to the Upstream repo - delete, update etc

3. Intermediate repositores do not keep the package - only 1- most-downstream repo which requested that package, and 2- the last upstream repo which connected to external repo/npm

### CodeArtifact Domain

1. Deduplicated storage - asset only needs to be stored once ina domain, even if its available in many repos (only pay once for storage)

2. Fast Copying

3. Easy Sharing - encrypted with same AWS EMS Key

### Activities

1. Create Repository with Upstream to external / npm / pypi store

2. Create Domain

3. Configure Connection instructions

4. Apply repository policy

## 1.5 CodeGuru

ML-powered service for:

1. automated Code Review - CodeGuru Reviewer

2. Application Performance (runtime) Recommendations - GodeGuru Profiler

Code (CodeGuru Reviewer) -> Build and Test (CodeGuru Profiler) -> deploy -> Measure ((CodeGuru Profiler))

### Extras

1. GodeGuru Reviewer Secrets Detector

    - identify hardcoded secrets in code

    - suggests remediation to protect secrets with Secrets Manager

2. Integrate CodeGuru Profiler to Lambda Functions

    - use Function Decorator `@with_lambda_profiler(profiling_group_name="myname")`

    - or Enable Profiling in Lambda function config

## 1.6 Image Builder

Automate the creation of Virtual Machines or Contianer images

Can run on a schedule - weekly, monthly etc.

Can publish AMI to multiple regions and multiple accounts

1. EC2 Image Builder -> Create Builder EC2 Instance (Build components applied / customize software on instance) -> New AMI -> Test EC2 Instance (Test suite is run) -> AMI is distributed (multi region, multi account)

2. CICD

-----------------------------------------
Code Pipeline

1. Build: CodeCommit and CodeBuild

2. Build AMI: AWS CloudFormation -> Triggers EC2 Image Builder -> New AMI

3. Rollout: CloudFormation - rolling update to ASG

### Extras

1. Use AWS RAM to share Images, Recipes, and Components across AWS accounts or thorugh AWS organization

2. Tracking latest AMIs - with SSM Parameter Store

    - EC2 Image Builder -> new AMI (with payload) -> notify SNS (invoke payload) -> Lambda (store ami-id) -> SSM Parameter Store

    - SSM Parameter Store can then be pulled by Users or CloudFormation



## 1.7 AWS Amplify

Web and mobile development tool. Elastic Beanstalk for web and mobile apps.

1. Configure Backend using Amplify CLI

    - Authentication, Storage, API, CICD, PubSub, AI/ML predictions

    - connect your source code from Github, CodeCommit, BitBucket or upload directly

2. Amplify Frontend Libraries - React, Vue, etc.. Flutter, iOS, etc..

3. Build and deploy with Amplify Console

### Extras

1. Connect CodeCommit and have one deployment per branch

    - dev branch, and prod branch
    
    - connect to AWS Amplify to create deployment

    - connect deployment to custom domain - dev.example.com

