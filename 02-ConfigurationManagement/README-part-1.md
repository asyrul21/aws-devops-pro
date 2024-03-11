# Domain 2: Configuration Management and Infra as Code: Part 1

## 2.1 CloudFormation

A declarative way of outlining your AWS Infrastructure for any resources, based on *CloudFormation Template*

Example (executed in order):

- I want a security group

- I want two EC2 instances using this security group

- I want two elastic IPs for these EC2 instances

- I want an S3 bucket

- I want a Load Balancer (ELB) in fornt of these EC2 instances

Benefits:

1. Infrastructure as code - no resources  are manually created

2. Infra code can be version controlled with Git

3. Changes to infra is reviewed through code

4. Can estimate infra cost based on resources defined in CloudFormation template

5. Can destroy and re-create infra on the fly

6. Automatic Diagram generation

7. Seperation of concerns

### Basic Flow

1. CloudFormation Template uploaded to S3

2. CloudFormation references the template file in (1)

3. A *CloudFormation Stack* is created, comprising of multiple AWS resources.

    - Stacks are identified by *name*.

    - Deleting a Stack deletes all every single Artifact that was created by CloudFormation

To update a template, you CANNOT edit a previous ones. Have upload a new version of the template.

### Deploying Templates

1. Manually

    - edit templates using a Code Editor or *CloudFormation Designer*

    - use console to input parameters, etc

    - suitable for learning

2. Automated way

    - Edit templates in a YAML file

    - Use AWS CLI to deploy templates, or use Continuous Delivery (CD) tool

### Building Blocks

1. Template

    - version

    - description

    - resources

    - paramteres

    - mappings

    - output

    - conditionals

2. Template Helpers

    - references

    - function

### Activities

1. Create a Stack

2. View stack in Designer

3. Click on *Physical ID* to go resource details page

4. Updating a Stack: 

 - Click Update > Replace Current Template

 - *Change Set Preview* - what will change

5. Deleting a Stack

  - CloudFormation > Delete

### CloudFormation Template

1. use `!Ref [Resource Name]` to reference them

```yml
---
# (optional) prompts users of some input value, during creation of stack
Parameters:
  SecurityGroupDescription:
    Description: Security Group Description
    Type: String

# (optional) define condition
Conditions:
  CreateProdResources: !Equals [ !Ref EnvType, prod]

# (optional) define mapping
Mappings:
  RegionMap:
    us-east-1:
      HVM64: ami-123123123
      HVMG2: ami-123123123
    us-west-1:
      HVM64: ami-123123123
      HVMG2: ami-123123123

Resources:
  # Resource Name
  MyInstance:
    # type of instance
    Type: AWS::EC2::Instance
    # instance configuration
    Properties:
      AvailabilityZone: us-east-1a
      ImageId: ami-a4c7edb2
      # or if use mapping: reference the map with Fn::FindInMap or !FindInMap
      ImageId: !FindInMap [RegionMap, !Ref "AWS::Region", HVM64]
      InstanceType: t2.micro
      SecurityGroups:
        # use !Ref [Resource Name] to reference them
        # !Ref is short for Fn::Ref
        - !Ref SSHSecurityGroup
        - !Ref ServerSecurityGroup
      # Deletion Policy
      DeletionPolicy: Delete

   # create an elastic IP for our instance
  MyEIP:
    Type: AWS::EC2::EIP
    Properties:
      InstanceId: !Ref MyInstance

  # EC2 security group
  SSHSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Enable SSH access via port 22
      SecurityGroupIngress:
      - CidrIp: 0.0.0.0/0
        FromPort: 22
        IpProtocol: tcp
        ToPort: 22

  # our second EC2 security group
  ServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: !Ref SecurityGroupDescription
      SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        CidrIp: 0.0.0.0/0
      - IpProtocol: tcp
        FromPort: 22
        ToPort: 22
        CidrIp: 192.168.1.1/32

  # Custom resource
  MyCustomResourceUsingLambda:
    # IMPORTANT
    Type: Custom: MyLambdaResource
    Properties:
      ServiceTokens: arn:aws:lambda:...
      # Input values (optional)
      ExampleProperty: "Example Value"

# exporting a value
Outputs:
  StackSSHSecurityGroup:
    Description: The SSH Security Group for our Company
    Value: !Ref MyCompanyWideSSHSecurityGroup
    # Export a Resource
    Export:
      Name: SSHSecurityGroup # must be unique in region
```

Referencing the output of another stack:

```yml
Resources:
  MySecureInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-123123123
      InstanceType: t2.micro
      AvailabilityZone: us-east-1a
      SecurityGroups:
      # Reference an output from another stack with Fn::ImportValue or !ImportValue
      - !ImportValue SSHSecurityGroup
```

### CloudFormation Template: Resources

Represent different AWS Components/Services that will be created and configured

Resource Type Identifier format: 

`service-provider::service-name::data-type-name`

eg.

`AWS::EC2::instance`

FAQs:

    1. Possible to create dynamic number of resources?

    - Yes, use CloudFormation *Macros and Transform*

    2. Are all Services supported?

    - Yes. Workaround that using *CloudFormation Custom Resources*

### CloudFormation Template: Parameters

Provides input to users.

1. When to use parameters?

- Is this CloudFormation resource configuration likely to change in the future?

- if so, make it a parameter

2. Settings:

  - Type: 
  
    - String
    
    - Number
    
    - Comma Delimeted List
    
    - List<Number>
    
    - AWS-Specific Parameter - help catch invalid values in the AWS Account

    - List<AWS-specific parameter>

    - SSM Parameter

    - ConstraintDescription

    - Min/MaxLength, Min/MaxValue

    - Default

    - AllowedValues (array/enum) - becomes a Dropdown

    ```yml
    Parameters:
      InstanceType:
        Description: Choose an EC2 instance type
        Type: String
        AllowedValues:
          - t2.micro
          - t2.small
          - t2.medium
        Default: t2.micro
    ```

    - AllowedPatterns (regex)

    - NoEcho


    ```yml
    Parameters:
      DBPassword:
        Description: The database admin password
        Type: String
        NoEcho: true
    ```

3. Pseudo Parameters - built in parameters

- AWS::AccountId -> 123123123

- AWS::Region -> us-east-1

- AWS::StackId

- AWS::StackName

- AWS::NotificationARNs

- AWS::NoValue

### CloudFormation Template: Mappings

Mappings are fixed variables within your CloudFormation template.

Important to differentiate different environments - dev vs prod, different regions, AMI types..

Works great for AMIs.

```yml
Mappings:
  RegionMap:
    us-east-1:
      HVM64: ami-123123123
      HVMG2: ami-123123123
    us-west-1:
      HVM64: ami-123123123
      HVMG2: ami-123123123

Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance
  Properties:
    # Reference the map with Fn::FindInMap or !FindInMap
    ImageId: !FindInMap [RegionMap, !Ref "AWS::Region", HVM64]
    instanceType: t2.micro
```

When to use Mappings instead of paramters:

- when you know in advance all the values that can be taken and that they can be deduced from variables such as:

  - Region

  - Availability Zone

  - AWS Account

  - Environment (dev vs prod)

- use paramters when the values are *user-specific*

### CloudFormation Template: Output

Declares optional output values that we can import into other stacks (need to export them first).

Useful if eg. you define a network CloudFormation, output the variables such as VPC ID and SubnetID.

Best way to perform some collaboration across stacks.

```yml
Outputs:
  StackSSHSecurityGroup:
    Description: The SSH Security Group for our Company
    Value: !Ref MyCompanyWideSSHSecurityGroup
    # Export a Resource
    Export:
      Name: SSHSecurityGroup # must be unique in region
```

To reference the output:

```yml
Resources:
  MySecureInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-123123123
      InstanceType: t2.micro
      AvailabilityZone: us-east-1a
      SecurityGroups:
      # Reference an output from another stack with Fn::ImportValue or !ImportValue
      - !ImportValue SSHSecurityGroup
```

IMPORTANT: If Stack A uses (import) an exported value from Stack B, you cannot delete Stack B until Stack A is deleted.

### CloudFormation Template: Conditions

Control creation of resources based on a condition.

```yaml
# define
Conditions:
  CreateProdResources: !Equals [ !Ref EnvType, prod]

# use
Resources:
  MountPoint:
    Type: AWS::EC2::VolumeAttachment
    Condition: CreateProdResources
```

1. Conditions:

  - Fn::And

  - Fn::Equals

  - Fn::If

  - Fn::Not

  - Fn::Or

### CloudFormation Template: Intrinsic Functions

1. Ref / !Ref

- reference parameters, resources

2. Fn::GetAtt / !GetAtt

- Get attributes attached to resources

- Check the docs to know what attributes a resource has. eg. [EC2](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-instance.html#aws-resource-ec2-instance-return-values-fn--getatt)

```yml
EBSVolume:
  Type: AWS::EC2::Volume
  Condition: CreateProdResources
  Properties:
    size: 100
    AvailabilityZone: !GetAtt EC2Instance.AvailabilityZone
```

3. Fn::FindInMap

- Get value in map

4. Fn::ImportValue

- import values that are exported in other stacks

5. Condition Functions (Fn::If, Fn::Not, Fn::Equals...)

6. Fn::Base64

- convert string to Base64 representation

- eg. pass encoded data to EC2 Instance's `UserData` property

```yml
Resources:
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ...
      UserData:
        Fn::Base64: |
          #!/bin/bash
          dnf update -y
          dnf install -y httpd
```

### CloudFormation Rollbacks

1. When Stack Creation Fails (Stack Failure Options):

 - default: everything rolls back (gets deleted) - can view in logs

 - can disable rollback and troubleshoot what happened


2. Update Fails

 - stack automatically rolls back to previous known working state - can view in logs

3. Rollback Failure - failure DURING rollback

 - Fix resources manually, then issue *ContinueUpdateRollbackAPI* from Console

    - use Console, or CLI or API Call

4. Activities:

  - Create Stack with Failures - see `trigger-failure.yml`

  - Creae an OK stack, but update (replace) with broken stack

### CloudFormation - Service Roles

IAM role that alllow CloudFormation to create/update/delete stack resources on your behalf.

Enables users to create/update/delete the stack resources even if they dont have permissions to work with the resources in the stack

Principle of least priviledge

```yml
iam:PassRole
```

Activity: 

1. AWS > IAM > Roles > Create Role > for AWS Service > choose CloudFormation > assign allowed Permissions

2. When creating stack, under Permissions > assign the role

### CloudFormation Capabilities

- `CAPABILITY_NAMED_IAM` (if resources are named) and `CAPABILITY_IAM`

  - necessary to enable when your CloudFormation template is creating or updating iAM resources (IAM User, Role, Group, Policy, Access Keys ...)

- CAPABILITY_AUTO_EXPAND

  - necessary when your CloudFormation template incudes Macros or Nested Stacks (stacks within stacks) to perform dynamic transformations

  - we are acknowledging that our template may change before deploying

- InsufficientCapabilitiesException

  - Exception thrown if capabilities havent been acknowldged when deploying a template

### CloudFormation DeletionPolicy Delete

1. DeletionPolicy:

  - Control what happens when the *CloudFormation template is deleted* or when *a resource is removed* from a CLoudFormation template

  - extra safety measure

2. Default: DeletionPolicy=*Delete*

```yml
DeletionPolicy: Delete
```

IMPORTANT: Delete wont work on an S3 Bucket if the bucket is not empty

3. DeletionPolicy *Retain* - specify on resources to preserve in case CloudFormation deletes

Supports all resources.

Status will be `DELETE_SKIPPED`

To delete it, *MANUALLY* delete it.

4. DeletionPolicy *Snapshot* - create one final snapshot before deleting the resource

Status will be `SNAPSHOT_CREATION`.

Supports EBS Volume, ElastiCache Cluster, ElastiCache ReplicationGroup

Can view the snapshots in EC2 > Elastic Block Store (EBS) > Snapshots

### CloudFormation Stack Policy

During a CloudFormation Stack Update, all update actions are allowed on all resources (default)

A Stack Policy is a JSON document that defines update actions that are allowed on specific resources during Stack updates

*When you set a Stack Policy, ALL resources in the Stack are protected by default*

YOu then need to explicitly *ALLOW* for the resources you want to be allowed to be updated.

- Protect resources from unintentional updates

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "Update:*",
      "Principal":"*",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "Update:*",
      "Principal":"*",
      "Resource": "LogicalResourceId/ProductionDatabase"
    }
  ]
}
```

### CloudFormation - TerminationProtection

Prevent accidental deletes of CloudFormation *Stacks*

After create your stack > Stacks Dashboard > Edit Termination Protection > Activate

### CloudFormation Custom Resources

- To define resources not yet supported by CloudFormation

- to define custom Provisioning Logic for resources outside of CLoudFormation

- if you want custom scripts run during create/update/delete through Lambda function

  - !!EXAM - eg. Running a Lambda function to empty an S3 bucket before being deleted

- default template using:

```
AWS::CloudFormation::CustomResource or
Custom::MyCustomResourceTypeName (recommended)
```

Backed by EITHER Lambda Function (most common) or SNS Topic.

```yml
Resources:
  MyCustomResourceUsingLambda:
    # IMPORTANT
    Type: Custom: MyLambdaResource
    Properties:
      # EITHER Lambda dunction ARN or SNS ARN - must be same region
      ServiceTokens: arn:aws:lambda:...
      # Input values (optional)
      ExampleProperty: "Example Value"
```

Use Case: Delete content from S3 bucket

- You cannot delete non-empty S3 bucket

- to delete a bucket, you much delete all objects inside it

- we can use Custom Resource to empty an S3 bucket before it gets deleted by CloudFormation

How it works:

  1. Developer creates a template with custom resource that;

  2. create/update/delete CloudFormation Stack

  3. Choose Resource Provider to backup the custom resource: Lambda or Amazon SNS

  4. Lambda or Amazon SNS will perform whatever you want via API Calls, then uploads JSON Response to S3 using *S3 Pre-Signed URL*

  5. CloudFormation listens to the S3

Activity:

  1. Create a Stack with Custom Resource with AWS Lambda to empty S3 Bucket

```yml
Parameters:
  S3BucketName:
    Type: String
    Description: "S3 bucket to create"
    AllowedPattern: "[a-zA-Z][a-zA-Z0-9_-]*"

Resources:
  SampleS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref S3BucketName
    DeletionPolicy: Delete

  S3CustomResource:
    Type: Custom::S3CustomResource
    Properties:
      ServiceToken: !GetAtt AWSLambdaFunction.Arn
      bucket_name: !Ref SampleS3Bucket    ## Additional parameter here

  AWSLambdaFunction:
     Type: AWS::Lambda::Function
     Properties:
       Description: "Empty an S3 bucket!"
       FunctionName: !Sub '${AWS::StackName}-${AWS::Region}-lambda'
       Handler: index.handler
       Role: !GetAtt AWSLambdaExecutionRole.Arn
       Timeout: 360
       Runtime: python3.8
       Code:
         ZipFile: |
          import boto3
          import cfnresponse
          ### cfnresponse module help in sending responses to CloudFormation
          ### instead of writing your own code

          def handler(event, context):
              # Get request type
              the_event = event['RequestType']        
              print("The event is: ", str(the_event))

              response_data = {}
              s3 = boto3.client('s3')

              # Retrieve parameters (bucket name)
              bucket_name = event['ResourceProperties']['bucket_name']
              
              try:
                  if the_event == 'Delete':
                      print("Deleting S3 content...")
                      b_operator = boto3.resource('s3')
                      b_operator.Bucket(str(bucket_name)).objects.all().delete()

                  # Everything OK... send the signal back
                  print("Execution succesfull!")
                  cfnresponse.send(event, context, cfnresponse.SUCCESS, response_data)
              except Exception as e:
                  print("Execution failed...")
                  print(str(e))
                  response_data['Data'] = str(e)
                  cfnresponse.send(event, context, cfnresponse.FAILED, response_data)

  AWSLambdaExecutionRole:
     Type: AWS::IAM::Role
     Properties:
       AssumeRolePolicyDocument:
         Statement:
         - Action:
           - sts:AssumeRole
           Effect: Allow
           Principal:
             Service:
             - lambda.amazonaws.com
         Version: '2012-10-17'
       Path: "/"
       Policies:
       - PolicyDocument:
           Statement:
           - Action:
             - logs:CreateLogGroup
             - logs:CreateLogStream
             - logs:PutLogEvents
             Effect: Allow
             Resource: arn:aws:logs:*:*:*
           Version: '2012-10-17'
         PolicyName: !Sub ${AWS::StackName}-${AWS::Region}-AWSLambda-CW
       - PolicyDocument:
           Statement:
           - Action:
             - s3:PutObject
             - s3:DeleteObject
             - s3:List*
             Effect: Allow
             Resource:
             - !Sub arn:aws:s3:::${SampleS3Bucket}
             - !Sub arn:aws:s3:::${SampleS3Bucket}/*
           Version: '2012-10-17'
         PolicyName: !Sub ${AWS::StackName}-${AWS::Region}-AWSLambda-S3
       RoleName: !Sub ${AWS::StackName}-${AWS::Region}-AWSLambdaExecutionRole
```

### CloudFormation Dynamic References

Reference external values stored in *Systems Manager Parameter Store* and *Secrets Manager* within CloudFormation templates

CloudFormation retireves the value of the specified reference during create/update/delete operations

eg: retrieve RDS DB Instance master password from Secrets Manager

Supports :

  - ssm - for plaintext values stored in SSM Parameter Store

    ```yml
    Resources:
      S3Bucket:
        Type: AWS::S3::Bucket
        Properties:
          AccessControl: '{{resolve:ssm:S3AccessControl:2}}'
    ```

  - ssm-secure - for secure strings stored in SSP Parameter Store

  ```yml
    Resources:
      IAMUser:
        Type: AWS::IAM::User
        Properties:
          UserName: John
          LoginProfile:
            Password: '{{resolve:ssm-secure:IAMUserPassword:10}}'
    ```

  - secretsmanager- for secret values stored in Secrets manager

  ```yml
    Resources:
      DBInstance:
        Type: AWS::RDS::DBInstance
        Properties:
          DBName: MyRDSInstance
          MasterUsername: '{{resolve:secretsmanager:MyRDSSecret:SecretString:username}}'
          MasterPassword: '{{resolve:secretsmanager:MyRDSSecret:SecretString:password}}'
    ```

  Syntax: `{{ resolve:service-name:reference-key }}`

  Integration (CloudFormation and RDS) Notes:

    1. Optionn 1: ManageMasterUserPassword

      - `ManageMasterUserPassword: true` - creates admin secret implicitly

        - RDS, and your selected DB Engine will manage the secret in SecretsManager and its rotation 

      - `Value: !GetAtt MyCluster.MasterUserSecret.SecretArn` - set in `output` of template 

    2. Option 2: Dynamic Reference

      1. Create a secret explicitly in CloudFormation template

      ```yml
      GenerateSecretString:
        SecretStringTemplate: '{"username": "admin" }'
        GenerateStringKey: "password",
        PasswordLength: 16,
        ExcludeCharacters: '"@/\'
      ```

      2. Reference secret in RDS DB Instance:

      ```yml
      MasterUsername: '{{resolve:secretsmanager:MyRDSSecret:SecretString:username}}'
      MasterPassword: '{{resolve:secretsmanager:MyRDSSecret:SecretString:password}}'
      ```

      3. Link the secret to RDS DB Instance for Rotation to work:

      ```yml
      SecretRDSAttachment:
        Type: AWS::SecretManager::SecretTargetAttachment
        Properties:
          SecretId: !Ref MyDatabaseSecret
          TargetId: !Ref MyDBInstance
          TargetType: AWS::RDS::DbInstance
      ```

### CloudFormation and EC2

We can have user data at EC2 instance launched through the console.

We can also write EC2 user-data script in out CloudFormation template.

*Pass the entire script through the function `Fn::Base64`*

See `cloudformation/10-cfn-init/1-ec2-user-data.yaml`

```yml
Resources:
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref ImageId
      InstanceType: t2.micro
      KeyName: !Ref KeyName
      SecurityGroups:
        - !Ref WebServerSecurityGroup
      UserData:
        Fn::Base64: |
           #!/bin/bash
           yum update -y
           amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2
           yum install -y httpd mariadb-server
           systemctl start httpd
           systemctl enable httpd
           usermod -a -G apache ec2-user
           chown -R ec2-user:apache /var/www
           chmod 2775 /var/www
           find /var/www -type d -exec sudo chmod 2775 {} \;
           find /var/www -type f -exec sudo chmod 0664 {} \;
           echo "<?php phpinfo(); ?>" > /var/www/html/phpinfo.php
```

You can view the logs by:

1. In CloudFormation, select the instance > Click Connect > *Connect using EC2 Instance Connect*

2. run `cat /var/log/cloud-init.log` or `cat /var/log/cfn-init.log`

### CloudFormatin Helper Scripts

Python scripts, that come directly on Amazon Linux AMIs, or can be installed using `yum` or `dnf` on non-Amazon Linux AMIs.

4 important scripts:

1. cfn-init: `AWS::CloudFormation::Init`

Used to retrieve and interpret the resource metadata, installing packages, creating files, and start services.

```yml

Resources:
  EC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ...
    # must be in metadata block
    Metadata:
      AWS::CloudFormation::Init:
        config:
          packages: 
          # to download and install pre-packaged apps and components
          # on Linux/Windows (eg. MySQL, PHP, etc)
          ...

          groups:
          # define user groups
          ...

          users:
          # define users, and which group they belong to
          ...

          sources:
          # download files and archives and place them in the EC2 instance
          ...

          files:
          # create files on the EC2 instance, using inline or
          # can be pulled from a URL
          ...

          commands:
          # run a series of commands
          ...

          services:
          # launch a list of sysvinit
          ...
```

From `cloudformation/10-cfn-init/2-cfn-init.yaml`:

```yml
WebServerHost:
    Type: AWS::EC2::Instance
    Metadata:
      Comment: Install a simple PHP application
      AWS::CloudFormation::Init:
        config:
          packages:
            yum:
              httpd: []
              php: []
          groups:
            apache: {}
          users:
            "apache":
              groups:
                - "apache"
          sources:
            "/home/ec2-user/aws-cli": "https://github.com/aws/aws-cli/tarball/master"
          files:
            "/tmp/cwlogs/apacheaccess.conf":
              content: !Sub |
                [general]
                state_file= /var/awslogs/agent-state
                [/var/log/httpd/access_log]
                file = /var/log/httpd/access_log
                log_group_name = ${AWS::StackName}
                log_stream_name = {instance_id}/apache.log
                datetime_format = %d/%b/%Y:%H:%M:%S
              mode: '000400'
              owner: apache
              group: apache
            "/var/www/html/index.php":
              content: !Sub |
                <?php
                echo '<h1>AWS CloudFormation sample PHP application for ${AWS::StackName}</h1>';
                ?>
              mode: '000644'
              owner: apache
              group: apache
            "/etc/cfn/cfn-hup.conf":
              content: !Sub |
                [main]
                stack=${AWS::StackId}
                region=${AWS::Region}
              mode: "000400"
              owner: "root"
              group: "root"
            "/etc/cfn/hooks.d/cfn-auto-reloader.conf":
              content: !Sub |
                [cfn-auto-reloader-hook]
                triggers=post.update
                path=Resources.WebServerHost.Metadata.AWS::CloudFormation::Init
                action=/opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource WebServerHost --region ${AWS::Region}
              mode: "000400"
              owner: "root"
              group: "root"
            # Fetch a webpage from a private S3 bucket
            "/var/www/html/webpage.html":
              source: !Sub "https://${MyS3BucketName}.s3.${AWS::Region}.amazonaws.com/webpage.html"
              mode: '000644'
              owner: apache
              group: apache
              authentication: S3AccessCreds
          commands:
            test:
              command: "echo \"$MAGIC\" > test.txt"
              env:
                MAGIC: "I come from the environment!"
              cwd: "~"
          services:
            sysvinit:
              httpd:
                enabled: 'true'
                ensureRunning: 'true'
              postfix:
                enabled: 'false'
                ensureRunning: 'false'
              cfn-hup:
                enable: 'true'
                ensureRunning: 'true'
                files:
                  - "/etc/cfn/cfn-hup.conf"
                  - "/etc/cfn/hooks.d/cfn-auto-reloader.conf"
      AWS::CloudFormation::Authentication:
        # Define S3 access credentials
        S3AccessCreds:
          type: S3
          buckets:
            - !Sub ${MyS3BucketName}
          roleName: !Ref InstanceRole
    CreationPolicy:
      ResourceSignal:
        Timeout: PT15M
        
    Properties:
      ImageId: !Ref ImageId
      KeyName: !Ref KeyName
      InstanceType: t2.micro
      IamInstanceProfile: !Ref InstanceProfile # Reference Instance Profile
      SecurityGroups:
      - !Ref WebServerSecurityGroup
      UserData:
        Fn::Base64:
          !Sub |
            #!/bin/bash -xe
            
            # Get the latest CloudFormation helper scripts
            yum install -y aws-cfn-bootstrap
            
            # Start cfn-init
            /opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource WebServerHost --region ${AWS::Region} || error_exit 'Failed to run cfn-init'
   
```

2. cfn-signal

A way to tell CloudFormation that the EC2 instance got properly configured after an `cfn-init` - by running cfn-signal script right after cfn-init

We define a *WaitCondition* -  block the template until it receives a signal from `cfn-signal`

- we attach a `CreationPolicy`

- we can define `Count > 1` if need more than 1 signal

```yml
UserData:
  Fn::Base64:
    !Sub |
      #!/bin/bash -xe
      
      # Get the latest CloudFormation helper scripts
      yum install -y aws-cfn-bootstrap
      
      # Start cfn-init
      /opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource WebServerHost --region ${AWS::Region}

      # ! Get result of last command
      INIT_STATUS=$?
      
      # cfn-init completed, send signal back using cfn-signal
      /opt/aws/bin/cfn-signal -e $INIT_STATUS --stack ${AWS::StackName} --resource WebServerHost --region ${AWS::Region} --resource SampleWaitCondition

      # exit
      exit $INIT_STATUS

SampleWaitCondition:
  CreationPolicy:
    ResourceSignal:
      Timeout: PT2M # 2 minutes
      Count: 1
  Type: AWS::CloudFormation::WaitCondition
```

- Potential Problems

  - WaitCondition Did not receive required Number of Signals from an Amazon EC2 Instance

    - ensur that the AMI you are using has the AWS CloudFormation *helper scripts* installed. If the AMI does not include the helper scripts, you can also download them to your instance

    - verify that the `cfn-init` and `cfn-signal` command was successfully run on the instance.

    - retrieve logs from `/var/log/cloud-init.log` or `/var/log/cfn-init.log`

    - verify the instance has connection to the internet: `curl -I https://aws.amazon.com`

- Handling failures

  - If failure happens, all the instances will be deleted

  - to disable this, change Rollback settings: Under *Stack failure options* > choose *Preserve successfully provisioned resources*


3. cfn-get-metadata

4. cfn- hup

### CloudFormation Nested Stacks

Stacks within another Stack. Considered as Best Practice.

Allow you to isolate repeated patterns/common components in seperate stacks and call them from other stacks.

Eg.:

1. Load Balancer Configuration that is reused

2. Security Group that is re-used

To *update* a nested stack, always update the parent (root) stack.

To *delete*, always delete the parent (root) stack.

Nested stacks can have nested stacks themselves

1. Cross Stacks vs Nester Stacks

  - Cross Stacks

    - helpful when stacks have different lifecycles

    - use Outputs Exports and `Fn::ImportValue`

    - pass values BETWEEN stacks (VPC id..)

  - Nested Stacks

    - Helpful when components must be reused.

    - example: re-use how to properly configure an Application Load Balancer

    - The nested stack only is important to the higher level stack - it is not shared


```yml
# Parameters to be passed to the nested Stack
Parameters:
  SSHKey:
    Type: AWS::EC2::KeyPair::KeyName
    Description: Name of an existing EC2 KeyPair to enable SSH access to the instance
  
Resources:
  myStack:
    # type is a Stack
    Type: AWS::CloudFormation::Stack
    Properties:
      # where to find the template
      # open the link below in Browser
      TemplateURL: https://s3.amazonaws.com/cloudformation-templates-us-east-1/LAMP_Single_Instance.template
      Parameters:
        KeyName: !Ref SSHKey
        DBName: "mydb"
        DBUser: "user"
        DBPassword: "pass"
        DBRootPassword: "passroot"
        InstanceType: t2.micro
        SSHLocation: "0.0.0.0/0"

Outputs:
  # reference of our stack
  StackRef:
    Value: !Ref myStack
  # reference output of our nested stack
  OutputFromNestedStack:
    Value: !GetAtt myStack.Outputs.WebsiteURL
```

2. Activiy

  - Create a nested stack

  - under *Capabilities* section, acknowledge *CAPABILITY_AUTO_EXPAND*

  - on left nav bar, toggle *View Nested*

### CloudFormation DependsOn

Specify creation of a specific resource follows another.

When added to a resource, that resource is created only after the creation of the depended resource.

IMPORTANT: Applied Automatically when using `!Ref` nad `!GetAtt`

```yml
Resources:
  EC2Instance:
    Type: AWS::EC2::Instance
    # depends on property
    DependsOn: DBInstance
    Properties:
      ImageId: 123123123
  DBInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      Engine: MySQL
      DBInstanceClass: db.t2.small
```

### CloudFormation StackSets (Extremely Expensive)

Create, update or delete stacks across multiple accounts and regions with a single operation/template.

You must have Administrator Account to create StackSets

Target accounts to create, delete, update, deelte stack instances from StackSets

When u update a stack set, all associated stack instances are updated throughout all accounts and regions

Can be applies to all accounts of an AWS organizations

1. Permission Models

- Self-managed Permissions - if not using AWS Organization

  - Create the IAM roles (with established trusted relationship) in both administrator and target accounts

  - deploy to any target account in which you have permission to create IAM role

- Service-managed Permissin - use AWS Organizations

  - Dont need to worry about IAM roles

  - Deploy to accounts managed by AWS Organizations

  - StackSets create the IAM roles on your behalf (enable trusted access with AWS Organizations)

  - must enable all features in AWS Organizations

  - Ability to deploy to accounts added to your organization in the future (Automatic Deployments)

  - Ability to *automatically deploy* Stack instances to new Accounts in an Organization


  - Can *delete* StackSets administration to member accounts in AWS Organization

  - *Trusted access with AWS Organizations* must be enabled before delegated admins can deploy to accounts managed by Organizations

2. Activity

  - Create a AdminRole Stack

    - this step is not required if using AWS Organizations

    ```yml
    AWSTemplateFormatVersion: 2010-09-09
    Description: Configure the AWSCloudFormationStackSetAdministrationRole to enable use of AWS CloudFormation StackSets.

    Resources:
      # Create the Administrato role
      AdministrationRole:
        Type: AWS::IAM::Role
        Properties:
          RoleName: AWSCloudFormationStackSetAdministrationRole
          AssumeRolePolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Principal:
                  Service: cloudformation.amazonaws.com
                Action:
                  - sts:AssumeRole
          Path: /
          Policies:
            - PolicyName: AssumeRole-AWSCloudFormationStackSetExecutionRole
              PolicyDocument:
                Version: 2012-10-17
                Statement:
                  - Effect: Allow
                    Action:
                      - sts:AssumeRole
                    Resource:
                      - "arn:*:iam::*:role/AWSCloudFormationStackSetExecutionRole"
    ```

  - Create Execution Role Stack

    - this step is not required if using AWS Organizations

    ```yml
    AWSTemplateFormatVersion: 2010-09-09
    Description: Configure the AWSCloudFormationStackSetExecutionRole to enable use of your account as a target account in AWS CloudFormation StackSets.

    Parameters:
      AdministratorAccountId:
        Type: String
        Description: AWS Account Id of the administrator account (the account in which StackSets will be created).
        MaxLength: 12
        MinLength: 12

    Resources:
      # Create execution role
      ExecutionRole:
        Type: AWS::IAM::Role
        Properties:
          RoleName: AWSCloudFormationStackSetExecutionRole
          AssumeRolePolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Principal:
                  AWS:
                    # reference the Administrator account your created in previous step
                    - !Ref AdministratorAccountId
                Action:
                  - sts:AssumeRole
          Path: /
          ManagedPolicyArns:
            - !Sub arn:${AWS::Partition}:iam::aws:policy/AdministratorAccess

    ```

  - Create StackSet

    - See `cloudformation/13-stacksets/enable-aws-config.yaml`

    - Under *Permissions*: Choose the Stack Set Admin role you created in previous step

    - Under *Accounts*: Choose whether to deploy stacks in *accounts* or Deploy stacks in *organization units*

    - specify Regions

    - Region COncurrency: Sequential or Parallel

  - Update a StackSet / Add a Stack to StackSet

    - In StackSets > Select your Stack Sets > Action > Add Stack to StackSet

  - Delete StackSet

    - You must first delete all Stacks within the StackSet

      - You can delete the Stack permanently or `Retain Stacks`

### CloudFormation Troubleshooting Stacks

1. DELETE_FAILED

  - Some resources must be emptied before deleting, such as S3 buckets

  - Use custom resources with Lambda functions to automate some actions. eg. empty a bucket before deleting

  - Security Groups cannot be deleted until all EC2 instances in the group are gone

  - Consider use DeletionPolicy=Retain to skip deletions

2. UPDATE_ROLLBACK_FAILED

  - can be caused by resources changed outside of CloudFormation, insufficient permissions, ASG does not receive enough signals

  - manually fix the error and then ContinueUpdateRollback

### CloudFormatin Troubleshooting StackSets

1. Stack instance status is OUTDATED

  - insufficient permissions ina target account for creating resources

  - template could be trying to create global resources that must be unique but are not, such as S3 buckets

  - Admin acount does not have a trust relationship with the target account

  - reached a limit or a quota in a target account (too many resources)

### CloudFormation Change Sets

When you update a stack, you need to know what changes will happen before applying them for greater confidence

ChangeSets wont say if the update will be successful.

For nested stacks, ChangeSets are applied to all child/nested stacks.

1. Activity:

  - Create Stack, but at bottom, Click *Create Change Set*

  - REVIEW_IN_PROGRESS

  - Go to ChangeSets tab > Select your ChangeSet > Execute

You will also find ChangeSets when *update* your Stacks

### CloudFormation Cfn-Hup

Can be used to tell your EC2 instance to look for *Metadata* changes (in your template file) every for eg. 15 minutes and apply the Metadata configuration again.

Relies on `cfn-hup` config file. See:

- /etc/cfn/cfn-hup.conf

- /etc/cfn/hooks.d/cfn-auto-reloader.conf

```yml
files:
  "/etc/cfn/cfn-hup.conf":
    content: !Sub |
      [main]
      stack=${AWS::StackId}
      region=${AWS::Region}
      interval=1 #minutes
  "/etc/cfn/hooks.d/cfn-auto-reloder.conf":
    content: !Sub |
      [cfn-auto-reloader-hook]
      triggers=post.update
      path=Resources.MyInstance.Metadata.AWS::CloudFormation::Init
      action=/opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource MyInstance --configSets default --region ${AWS::Region} runas=root
```

### CloudFormation Drift

Drift: when your infrastructure resources has been manually changed, updated (from CLI for eg.).

To detect drifts.

- Can detect drifts on entire stack or individual resource within a stack

- StackSet Drift detection - can be stopped/enabled

  - Perform drift detection on the stack associated with each stack instance in StackSet

  - Drift detection identifies unmanaged changes (from outside of CloudFormation)

  - Changes made through CloudFormation to a Stack (not StackSet) are NOT considered as drifted

Go to *Stck Actions* > Detect Drift

Go to *Stack Actions* > View Drift Result