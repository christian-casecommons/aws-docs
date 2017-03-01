---
date: 2017-02-20T06:39:45+13:00
title: Web Proxy Stack
weight: 60
---

## Introduction

The [aws-cloudformation](https://github.com/casecommons/aws-cloudformation) role includes a [web proxy template](https://github.com/casecommons/aws-cloudformation/blob/master/templates/proxy.yml.j2) that is designed to create the following AWS resources:

- Web Proxy using the popular [squid](http://www.squid-cache.org) open source web proxy application
- EC2 autoscaling group for running ECS container instances
- ECS cluster including ECS container instances that can run the squid containers
- ECS task definition for defining the runtime configuration of the squid container
- ECS service for running the squid containers as a long running service, integrated with a front-end Elastic Load Balancer (ELB)
- Elastic Load Balancer (ELB) for load balancing incoming HTTP proxy requests from hosts using the squid proxy
- Route 53 host record defining a friendly URL for the squid proxy

{{% note title="Note" %}}
Before commencing the tasks below, ensure you have successfully completed all tasks in the following sections:

- [Security Resources]({{< relref "security-resources/index.md" >}})
- [CloudFormation Resources]({{< relref "cloudformation-resources/index.md" >}})
- [EC2 Container Registry Resources]({{< relref "ecr-resources/index.md" >}})
{{% /note %}}

## Creating an ECS AMI

We are creating our first CloudFormation stack that includes ECS resources, and we need to create a custom ECS Amazon Machine Image (AMI) that is designed to work with the proxy template.

The Packer ECS repository is located at https://github.com/Casecommons/packer-ecs and includes a workflow that can be configured to publish the Packer ECS image to the AWS account of your choice.

1\. Clone the Packer ECS repository to your local environment:

```bash
$ git clone git@github.com:casecommons/packer-ecs
Cloning into 'packer-ecs'...
remote: Counting objects: 143, done.
remote: Total 143 (delta 0), reused 0 (delta 0), pack-reused 143
Receiving objects: 100% (143/143), 21.66 KiB | 0 bytes/s, done.
Resolving deltas: 100% (55/55), done.
$ cd packer-ecs
$ tree -L 1
.
├── Makefile
├── Makefile.settings
├── README.md
├── docker
└── src
```

2\. Open the `Makefile` at the root of the **packer-ecs** repository and modify the highlighted settings:

{{< highlight make "hl_lines=5" >}}
# Project variables
export PROJECT_NAME ?= packer

# AWS security settings
AWS_ROLE ?= arn:aws:iam::160775127577:role/admin
AWS_SG_NAME ?= packer-$(MY_IP_ADDRESS)-$(TIMESTAMP)
AWS_SG_DESCRIPTION ?= "Temporary security group for Packer"
...
...
{{< /highlight >}}

Here we ensure the `Makefile` settings are configured with the IAM **admin** role for the **demo-resources** account (`AWS_ROLE`), as the ECS AMI build process will publish the AMI to the account associated with this role.

3\. Run the `make release` command to build the Packer image:

{{< highlight bash "hl_lines=37" >}}
$ export AWS_PROFILE=demo-resources-admin
$ make release
Enter MFA code: ******
=> Starting packer build...
=> Creating packer security group...
=> Creating packer image...
Building packer
Step 1/10 : FROM alpine
...
...
Step 10/10 : CMD packer build packer.json
 ---> Running in 7de75cd744a5
 ---> 6cefa9dcc744
Removing intermediate container 7de75cd744a5
Successfully built 6cefa9dcc744
=> Running packer build...
Creating network "packer_default" with the default driver
Creating packer_packer_1
Attaching to packer_packer_1
packer_1  | 2017-02-26T09:52:22Z 52bf8983fb4e confd[7]: INFO Backend set to env
packer_1  | 2017-02-26T09:52:22Z 52bf8983fb4e confd[7]: INFO Starting confd
packer_1  | 2017-02-26T09:52:22Z 52bf8983fb4e confd[7]: INFO Backend nodes set to
packer_1  | 2017-02-26T09:52:22Z 52bf8983fb4e confd[7]: INFO Target config /packer/packer.json out of sync
packer_1  | 2017-02-26T09:52:22Z 52bf8983fb4e confd[7]: INFO Target config /packer/packer.json has been updated
packer_1  | amazon-ebs output will be in this color.
packer_1  |
packer_1  | ==> amazon-ebs: Prevalidating AMI Name...
packer_1  |     amazon-ebs: Found Image ID: ami-8e7bc4ee
packer_1  | ==> amazon-ebs: Creating temporary keypair: packer_58b2a556-84ff-72db-9874-eb14567c2bc5
packer_1  | ==> amazon-ebs: Launching a source AWS instance...
packer_1  |     amazon-ebs: Instance ID: i-01c267d7e0219e85f
...
...
packer_1  | ==> Builds finished. The artifacts of successful builds are:
packer_1  | --> amazon-ebs: AMIs were created:
packer_1  |
packer_1  | us-west-2: ami-4662e126
packer_1  | --> amazon-ebs:
packer_packer_1 exited with code 0
=> Deleting packer security group...
=> Deleted packer security group...
=> Build complete
{{< /highlight >}}

The ECS AMI build will take a few minutes to complete.  Once the build has finished, take a note of the AMI ID, which in the output above is **ami-4662e126** 

4\. Run the `make clean` command to clean up the local Docker environment.

```bash
$ make clean
=> Destroying release environment...
=> Removing dangling images...
=> Clean complete
```

5\. In the AWS console, navigate to **EC2 > Images > AMIs**:

![Squid Image](/images/ecs-ami.png)

Notice that the AMI built in Step 3 is published and available in the **demo-resources** account.

## Creating an EC2 Key Pair

The web proxy template requires an EC2 keypair name to be provided as an input, which will be granted SSH access to the ECS container instances created as part of the web proxy stack.

1\. In the AWS console, navigate to **EC2 > Network & Security > Key Pairs** and click on the **Create Key Pair** button:

![EC2 Key Pairs](/images/ec2-key-pairs.png)

2\. Name the key **admin** and click on the **Create** button.

![Create EC2 Key Pair](/images/ec2-create-key-pair.png)

3\. A new key pair is created and a SSH private key file is downloaded to your computer:

![Create EC2 Key Pair](/images/ec2-admin-key-pair.png)

4\. Move the SSH private key file to your `~/.ssh` folder and ensure the required permissions are set:

```bash
$ mv ~/Downloads/admin.pem.txt ~/.ssh/demo-resources-admin.pem
$ chmod 600 ~/.ssh/demo-resources-admin.pem
```

## Creating the Playbook

We can now get started establishing a proxy playbook that defines a proxy stack for the **Demo Resources** account.

1\. Clone the [AWS Starter](https://github.com/casecommons/aws-starter) to a local folder called `demo-proxy` and the re-initialise the Git repository.

```bash
$ git clone git@github.com:casecommons/aws-starter.git demo-proxy
  Cloning into 'demo-proxy'...
  remote: Counting objects: 22, done.
  remote: Compressing objects: 100% (14/14), done.
  remote: Total 22 (delta 4), reused 22 (delta 4), pack-reused 0
  Receiving objects: 100% (22/22), done.
  Resolving deltas: 100% (4/4), done
$ cd demo-proxy
$ rm -rf .git
$ git init
Initialized empty Git repository in /Users/jmenga/Source/casecommons/demo-proxy/.git/
```

2\.  Update the `roles/requirements.yml` file to the desired versions for each role:

{{< highlight python "hl_lines=4 8" >}}
# You can specify git tag, commit or branch for the version property
- src: git@github.com:casecommons/aws-cloudformation.git
  scm: git
  version: 0.7.0
  name: aws-cloudformation
- src: git@github.com:casecommons/aws-sts.git
  scm: git
  version: 0.1.2
  name: aws-sts
{{< /highlight >}}

3\.  Install the roles using the `ansible-galaxy` command as demonstrated below:

{{< highlight python >}}
$ ansible-galaxy install -r roles/requirements.yml --force
- extracting aws-cloudformation to /Users/jmenga/Source/casecommons/demo-proxy/roles/aws-cloudformation
- aws-cloudformation was installed successfully
- extracting aws-sts to /Users/jmenga/Source/casecommons/demo-proxy/roles/aws-sts
- aws-sts was installed successfully
{{< /highlight >}}

4\.  Modify the `group_vars/all/vars.yml` file, which contains global settings for the playbook:

{{< highlight python "hl_lines=3" >}}
# Stack Settings
cf_stack_name: web-proxy
cf_stack_template: "templates/proxy.yml.j2"
cf_stack_tags:
  org:business:owner: Casecommons
  org:business:product: Web Proxy
  org:business:severity: High
  org:tech:environment: "{{ env }}"
  org:tech:contact: pema@casecommons.org

# Stack inputs
cf_stack_inputs:
  ApplicationImageId: "{{ config_application_ami }}"
  ApplicationInstanceType: "{{ config_application_instance_type }}"
  ApplicationAutoscalingDesiredCount: "{{ config_application_desired_count | default(config_az_count) | default(2) | int }}"
  KeyName: "{{ config_application_keyname }}"
  Environment: "{{ env }}"
  ProxyImage: "{{ config_proxy_image }}"
  ProxyImageTag: "{{ config_proxy_tag }}"
  ProxyDisableWhitelist: "{{ config_proxy_disable_whitelist | default(false) | string | lower }}"
  ProxyWhitelist: "{{ config_proxy_whitelist | default('') }}"
{{< /highlight >}}

Notice that we reference the [templates/proxy.yml.y2 template](https://github.com/casecommons/aws-cloudformation/blob/master/templates/proxy.yml.j2) that is embedded within the [aws-cloudformation role](https://github.com/casecommons/aws-cloudformation).

We also add a dictionary called `cf_stack_inputs`, which provides values for input parameters defined in the proxy CloudFormation template.  

Notice that these settings reference environment specific settings prefixed with `config_`, which will we need to define in our environment settings.

5\. Remove the local `templates` folder, since we are using a template that is embedded within the `aws-cloudformation` role:

```bash
$ rm -rf templates
```

## Defining a New Environment

We will now add a new environment called **demo-resources** to the playbook, which will be used to create CloudFormation resources in the **demo-resources** account.

1\. Modify the `inventory` file so that it defines a single environment called **demo-resources**, ensuring you specify `ansible_connection=local`:

```toml
[demo-resources]
demo-resources ansible_connection=local
```

2\. Remove the `group_vars/non-prod` environment folder that is included in the starter template:

```bash
$ rm -rf group_vars/non-prod
```

3\. Create a file called `group_vars/demo-resources/vars.yml`, which will hold all environment specific configuration for the **demo-resources** environment:

```bash
$ mkdir -p group_vars/demo-resources
$ touch group_vars/demo-resources/vars.yml
```

4\. Add the following settings to `group_vars/demo-resources/vars.yml`:

```python
# STS role in the target AWS account to assume
sts_role_arn: "arn:aws:iam::160775127577:role/admin"

# Application settings
config_application_keyname: admin
config_application_instance_type: t2.micro
config_application_ami: ami-4662e126
config_az_count: 3
config_log_deletion_policy: Delete

# Proxy settings
config_proxy_image: 160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/squid
config_proxy_tag: latest
config_proxy_disable_whitelist: false
config_proxy_whitelist: .demo.cloudhotspot.co
config_proxy_network_mode: host
```

Here we target the **demo-resources** account by specifying the **demo-resources** IAM **admin** role in the `sts_role_arn` variable, whilst the remaining settings configure the proxy template:

- `config_application_keyname` - specifies the name of the EC2 key pair that ECS container instances will be created with.  Notice this matches the name of the [EC2 key pair created earlier]({{< relref "web-proxy/index.md#creating-an-ec2-key-pair" >}}). 
- `config_application_instance_type` - specifies the EC2 instance type for the ECS container instances that will be created.
- `config_application_ami` - specifies the AMI ID of the image used to create the ECS container instances.  Notice this matches the ID of the [AMI created earlier]({{< relref "web-proxy/index.md#creating-an-ecs-ami" >}})
- `config_az_count` - specifies the number of availability zones to place ECS conatiner instances into.
- `config_log_deletion_policy` - specifies whether to "Delete" or "Retain" CloudWatch log groups when the stack is destroyed.
- `config_proxy_image` - specifies the Docker image used to run the squid proxy containers.  Notice this matches the image created in [EC2 Container Registry Resources]({{< relref "ecr-resources/index.md#publishing-docker-images" >}})
- `config_proxy_tag` - specifies the Docker image tag used to run the squid proxy containers.
- `config_proxy_disable_whitelist` - disables the web proxy whitelist when set to `true`.  The default setting is `false`.
- `config_proxy_whitelist` - defines domains that will be added to the web proxy whitelist.  Here we add `.demo.cloudhotspot.co`, which means any host in `*.demo.cloudhotspot.co` will be whitelisted.
- `config_proxy_network_mode` - configures the Docker networking mode - in this example it is set to `host`, meaning the squid container will share the ECS container instance operating system network stack.

{{< note title="Note" >}}
The squid Docker image created in [EC2 Container Registry Resources]({{< relref "ecr-resources/index.md#publishing-docker-images" >}}) includes a whitelist that only permits access to AWS service API endpoints. See https://github.com/Casecommons/docker-squid#squid-whitelist for details on the default whitelist.
{{< /note >}}

## Running the Playbook

Now that we've defined environment settings for the **demo-resources** environment, let's run the playbook to create ECR repository resources for the environment.  

1\. Ensure your local AWS environment is configured to target the **demo-resources** account:

```bash
$ export AWS_PROFILE=demo-resources-admin
```

2\. Run the Ansible playbook targeting the `demo-resources` environment as demonstrated below:

{{< highlight bash >}}
$ ansible-playbook site.yml -e env=demo-resources

PLAY [Assume Role] *************************************************************

TASK [aws-sts : set_fact] ******************************************************
ok: [demo-resources]

TASK [aws-sts : checking if sts functions are sts_disabled] ********************
skipping: [demo-resources]

TASK [aws-sts : setting empty sts_session_output result] ***********************
skipping: [demo-resources]

TASK [aws-sts : setting sts_creds if legacy AWS credentials are present (e.g. for Ansible Tower)] ***
skipping: [demo-resources]

TASK [aws-sts : assume sts role] ***********************************************
Enter MFA code: ******
ok: [demo-resources]
...
...
TASK [aws-cloudformation : set local path fact if s3 upload disabled] **********
ok: [demo-resources]

TASK [aws-cloudformation : configure application stack] ************************
...
...
PLAY RECAP *********************************************************************
demo-resources             : ok=20   changed=1    unreachable=0    failed=0
{{< /highlight >}}

3\.  The playbook will take approxiately 10 minutes to create the CloudFormation stack and associated resources.  Whilst the CloudFormation stack is being created, you can review the CloudFormation stack that was generated in the `build/<timestamp>` folder:

```bash
$ tree build
build
└── 20170227015822
    ├── web-proxy-config.json
    ├── web-proxy-policy.json
    ├── web-proxy-stack.json
    └── web-proxy-stack.yml
```

The following shows the `web-proxy-stack.yml` file that was generated and uploaded to CloudFormation:

```python
AWSTemplateFormatVersion: "2010-09-09"

Description: Squid Forward Proxy

Parameters:
  ApplicationImageId:
    Type: String
    Description: Application Amazon Machine Image ID
  ApplicationInstanceType:
    Type: String
    Description: Application EC2 Instance Type
  ApplicationAutoscalingDesiredCount:
    Type: Number
    Description: Application AutoScaling Group Desired Count
    Default: 3      
  ProxyImage:
    Type: String
    Description: Docker Image for Proxy
    Default: 429614120872.dkr.ecr.us-west-2.amazonaws.com/cwds/squid
  ProxyImageTag:
    Type: String
    Description: Docker Image Tag for Proxy
    Default: latest
  ProxyDisableWhitelist:
    Type: String
    Description: Disables whitelist when set to true.  Proxy will operate as an open proxy.
    Default: "false"
    AllowedValues:
    - "true"
    - "false"
  ProxyWhitelist:
    Type: String
    Description: Comma separated list of domains to whitelist
    Default: ""
  KeyName:
    Type: String
    Description: EC2 Key Pair for SSH Access
  Environment:
    Type: String
    Description: Stack Environment
Resources:
  ProxyDnsRecord:
    Type: "AWS::Route53::RecordSet"
    Properties:
      Name:
        Fn::Join: ["", [ "proxy.", "Fn::ImportValue": "DefaultVpcDomain" ] ]
      TTL: "300"
      HostedZoneName:
        Fn::Join: ["", [ "Fn::ImportValue": "DefaultVpcDomain", "." ] ]
      Type: "CNAME"
      Comment: 
        Fn::Sub: "Forward Proxy - ${Environment}"
      ResourceRecords:
        - Fn::Sub: "${ApplicationLoadBalancer.DNSName}"
  ApplicationLoadBalancer:
    Type : "AWS::ElasticLoadBalancing::LoadBalancer"
    Properties:
      Scheme: "internal"
      SecurityGroups:
        - Ref: "ApplicationLoadBalancerSecurityGroup"
      Subnets:
        - Fn::ImportValue: "DefaultPublicSubnetA"
        - Fn::ImportValue: "DefaultPublicSubnetB"
        - Fn::ImportValue: "DefaultPublicSubnetC"
      CrossZone: "true"
      ConnectionSettings:
        IdleTimeout: 120
      ConnectionDrainingPolicy:
        Enabled: "true"
        Timeout: 120
      Listeners:
        - LoadBalancerPort: "3128"
          InstancePort: "3128"
          Protocol: "tcp"
      Tags:
        - Key: Name
          Value:
            Fn::Sub: ${AWS::StackName}-${Environment}-elb
  ApplicationLoadBalancerSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      VpcId:
        Fn::ImportValue: "DefaultVpcId"
      GroupDescription: "Proxy Load Balancer Security Group"
      SecurityGroupIngress:
        - IpProtocol: "tcp"
          FromPort: 3128
          ToPort: 3128
          CidrIp:
            Fn::ImportValue: "DefaultVpcCidr"
  ApplicationLoadBalancerToAutoscalingIngressTCP3128:
    Type: "AWS::EC2::SecurityGroupIngress"
    Properties:
      IpProtocol: "tcp"
      FromPort: 3128
      ToPort: 3128
      GroupId: { "Ref": "ApplicationAutoscalingSecurityGroup" }
      SourceSecurityGroupId: { "Ref": "ApplicationLoadBalancerSecurityGroup" }
  ApplicationLoadBalancerToAutoscalingEgressTCP3128:
    Type: "AWS::EC2::SecurityGroupEgress"
    Properties:
      IpProtocol: "tcp"
      FromPort: 3128
      ToPort: 3128
      GroupId: { "Ref": "ApplicationLoadBalancerSecurityGroup" }
      DestinationSecurityGroupId: { "Ref": "ApplicationAutoscalingSecurityGroup" }
  ApplicationAutoscaling:
    Type: "AWS::AutoScaling::AutoScalingGroup"
    DependsOn:
      - DmesgLogGroup
      - DockerLogGroup
      - EcsAgentLogGroup
      - EcsInitLogGroup
      - MessagesLogGroup
    CreationPolicy:
      ResourceSignal:
        Count: { "Ref": "ApplicationAutoscalingDesiredCount"}
        Timeout: "PT15M"
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: { "Ref": "ApplicationAutoscalingDesiredCount"}
        MinSuccessfulInstancesPercent: "100"
        WaitOnResourceSignals: "true"
        PauseTime: "PT15M"
    Properties:
      VPCZoneIdentifier:
        - Fn::ImportValue: "DefaultPublicSubnetA"
        - Fn::ImportValue: "DefaultPublicSubnetB"
        - Fn::ImportValue: "DefaultPublicSubnetC"
      LaunchConfigurationName: { "Ref" : "ApplicationAutoscalingLaunchConfiguration" }
      MinSize: "0"
      MaxSize: "4"
      DesiredCapacity: { "Ref": "ApplicationAutoscalingDesiredCount" }
      Tags:
        - Key: Name
          Value:
            Fn::Sub: ${AWS::StackName}-instance
          PropagateAtLaunch: "true"
  ApplicationAutoscalingLaunchConfiguration:
    Type: "AWS::AutoScaling::LaunchConfiguration"
    Metadata:
      AWS::CloudFormation::Init:
        config:
          commands:
            10_first_run:
              command: "sh firstrun.sh"
              env:
                STACK_NAME: { "Ref": "AWS::StackName" }
                AUTOSCALING_GROUP: "ApplicationAutoscaling"
                AWS_DEFAULT_REGION: { "Ref": "AWS::Region" }
                ECS_CLUSTER: { "Ref": "ApplicationCluster" }
                DOCKER_NETWORK_MODE: "host"
              cwd: "/home/ec2-user/"
          files:
            /etc/ecs/ecs.config:
              content: 
                Fn::Sub: "ECS_CLUSTER=${ApplicationCluster}\n"
    Properties:
      ImageId: { "Ref": "ApplicationImageId" }
      InstanceType: { "Ref": "ApplicationInstanceType" }
      AssociatePublicIpAddress: "true"
      IamInstanceProfile: { "Ref": "ApplicationAutoscalingInstanceProfile" }
      KeyName: { "Ref": "KeyName" }
      SecurityGroups:
        - Ref: "ApplicationAutoscalingSecurityGroup"
      UserData: 
        Fn::Base64:
          Fn::Join: ["\n", [
            "#!/bin/bash",
            "set -e",
            "Fn::Sub": "/opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource ApplicationAutoscalingLaunchConfiguration --region ${AWS::Region}",
            "Fn::Sub": "/opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource ApplicationAutoscaling --region ${AWS::Region}"
          ] ]
  ApplicationAutoscalingSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      VpcId:
        Fn::ImportValue: "DefaultVpcId"
      GroupDescription: "Proxy Security Group"
      SecurityGroupIngress:
        - IpProtocol: "tcp"
          FromPort: 22
          ToPort: 22
          CidrIp:
            Fn::ImportValue: "DefaultManagementSubnetACidr"
        - IpProtocol: "tcp"
          FromPort: 22
          ToPort: 22
          CidrIp:
            Fn::ImportValue: "DefaultManagementSubnetBCidr"
      SecurityGroupEgress:
        - IpProtocol: "tcp"
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: "tcp"
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
        - IpProtocol: "udp"
          FromPort: 53
          ToPort: 53
          CidrIp:
            Fn::Join: ["", [
              "Fn::ImportValue": "DefaultVpcDnsServer", "/32"
            ] ]
        - IpProtocol: "udp"
          FromPort: 123
          ToPort: 123
          CidrIp: 0.0.0.0/0
  ApplicationAutoscalingRole:
    Type: "AWS::IAM::Role"
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: "Allow"
            Principal:
              Service: [ "ec2.amazonaws.com" ]
            Action: [ "sts:AssumeRole" ]
      Path: "/"
      ManagedPolicyArns: []
      Policies:
        - {"PolicyName": "CloudWatchLogs", "PolicyDocument": {"Version": "2012-10-17", "Statement": [{"Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents", "logs:DescribeLogStreams"], "Resource": {"Fn::Sub": "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:${AWS::StackName}*"}, "Effect": "Allow"}]}}
        - PolicyName: "EC2ContainerInstancePolicy"
          PolicyDocument:
            Statement:
              - Effect: "Allow"
                Action:
                  - "ecs:RegisterContainerInstance"
                  - "ecs:DeregisterContainerInstance"
                Resource: 
                  Fn::Sub: "arn:aws:ecs:${AWS::Region}:${AWS::AccountId}:cluster/${ApplicationCluster}"
              - Effect: "Allow"
                Action:
                  - "ecs:DiscoverPollEndpoint"
                  - "ecs:Submit*"
                  - "ecs:Poll"
                  - "ecs:StartTelemetrySession"
                Resource: "*"
              - Effect: "Allow"
                Action: 
                  - "ecr:BatchCheckLayerAvailability"
                  - "ecr:BatchGetImage"
                  - "ecr:GetDownloadUrlForLayer"
                  - "ecr:GetAuthorizationToken"
                Resource: "*"
  ApplicationAutoscalingInstanceProfile:
    Type: "AWS::IAM::InstanceProfile"
    Properties:
      Path: "/"
      Roles: [ { "Ref": "ApplicationAutoscalingRole" } ]
  ApplicationCluster:
    Type: "AWS::ECS::Cluster"
  ApplicationTaskDefinition:
    Type: "AWS::ECS::TaskDefinition"
    Properties:
      NetworkMode: "host"
      ContainerDefinitions:
      - Name: squid
        Image:
          Fn::Sub: ${ProxyImage}:${ProxyImageTag}
        Memory: 995
        LogConfiguration:
          LogDriver: awslogs
          Options:
            awslogs-group: 
              Fn::Sub: ${AWS::StackName}/ecs/ProxyService/squid
            awslogs-region: { "Ref": "AWS::Region" }
        PortMappings:
          - ContainerPort: 3128
            Protocol: tcp
        Environment:
        - Name: NO_WHITELIST
          Value: { "Ref": "ProxyDisableWhitelist" }
        - Name: SQUID_WHITELIST
          Value: { "Ref": "ProxyWhitelist" }
  ProxyService:
    Type: "AWS::ECS::Service"
    DependsOn: 
      - ApplicationAutoscaling
      - ProxyServiceLogGroup
    Properties:
      Cluster: { "Ref": "ApplicationCluster" }
      TaskDefinition: { "Ref": "ApplicationTaskDefinition" }
      DesiredCount: { "Ref": "ApplicationAutoscalingDesiredCount"}
      DeploymentConfiguration:
          MinimumHealthyPercent: 50
          MaximumPercent: 200
      LoadBalancers:
        - ContainerName: "squid"
          ContainerPort: "3128"
          LoadBalancerName: { "Ref": "ApplicationLoadBalancer" }
      Role: { "Ref": "EcsServiceRole" }
  EcsServiceRole:
    Type: "AWS::IAM::Role"
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: "Allow"
            Principal: 
              Service: [ "ecs.amazonaws.com" ]
            Action: [ "sts:AssumeRole" ]
      Path: "/"
      ManagedPolicyArns:
        - "arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceRole"
  DmesgLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName: 
        Fn::Sub: ${AWS::StackName}/ec2/ApplicationAutoscaling/var/log/dmesg
      RetentionInDays: 30
  DockerLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName: 
        Fn::Sub: ${AWS::StackName}/ec2/ApplicationAutoscaling/var/log/docker
      RetentionInDays: 30
  EcsAgentLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName: 
        Fn::Sub: ${AWS::StackName}/ec2/ApplicationAutoscaling/var/log/ecs/ecs-agent
      RetentionInDays: 30
  EcsInitLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName: 
        Fn::Sub: ${AWS::StackName}/ec2/ApplicationAutoscaling/var/log/ecs/ecs-init
      RetentionInDays: 30
  MessagesLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName: 
        Fn::Sub: ${AWS::StackName}/ec2/ApplicationAutoscaling/var/log/messages
      RetentionInDays: 30
  ProxyServiceLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ecs/ProxyService/squid
      RetentionInDays: 30
   
Outputs:
  ProxyURL:
    Description: "Squid Proxy URL"
    Value:
      Fn::Join: ["", [
        "http://proxy.",
        "Fn::ImportValue": "DefaultVpcDomain",
        ":3128"
      ] ]
    Export:
      Name: "DefaultProxyURL"
  ProxySecurityGroup:
    Description: "Squid Proxy Security Group"
    Value:
      Ref: ApplicationLoadBalancerSecurityGroup
    Export:
      Name: "DefaultProxySecurityGroup"
```

4\. Once the playbook execution completes successfully, login to the **demo-resources** account.  You should see a new CloudFormation stack called `web-proxy`:

![Web Proxy Stack](/images/web-proxy.png)

5\. If you open the ECS dashboard and select **Clusters**, you can see a single ECS cluster was created:

![Web Proxy ECS Cluster](/images/web-proxy-ecs-cluster.png)

## Wrap Up

We created a web proxy in our **demo-resources** account, which provides the ability for hosts located on private subnets to be able to communicate securely with AWS APIs.

{{< note title="Note" >}}
At this point you should commit your changes to the web proxy playbook before continuing.
{{< /note >}}
