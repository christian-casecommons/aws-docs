---
date: 2017-02-27T02:59:22+13:00
title: Intake API Stack
weight: 70
---

## Introduction

At this point, we have established all of the shared resource stacks in our **demo-users** and **demo-resources** accounts, and it is now time to focus on defining a playbook and CloudFormation stack for an application.

We will now deploy the Intake API application to our **demo-resources** account, creating two environments:

- `dev` - development environment intended for continuous deployment
- `prod` - production environment with controlled releases

In this tutorial, the Intake API application will be published at the following HTTPS endpoints:

- `https://dev-intake-api.demo.cloudhotspot.co` - development environment endpoint
- `https://intake-api.demo.cloudhotspot.co` - production environment endpoint

## Configuring DNS Delegation

Before we can create public DNS endpoints for the Intake API stack, we must ensure that DNS delegation is configured for the `demo.cloudhotspot.co` domain we created earlier in the [Network Resources]({{< relref "network-resources/index.md" >}}) section.

In this example, the parent domain `cloudhotspot.co` is hosted on CloudFlare, so we must configure DNS delegation records on the parent domain, delegating requests for `*.demo.cloudhotspot.co` to the AWS Route 53 service.

1\. In the AWS Console, navigate to **Route53 > Hosted zones**:

![Route53 Domains](/images/network-domains.png)

2\. Select the `demo.cloudhotspot.co` domain and copy each of the **NS** record values:

![Route53 Name Servers](/images/route53-name-servers.png)

3\. Here is an example of configuring CloudFlare **NS** records for the `demo` child domain of the `cloudhotspot.co` root domain:

![CloudFlare Delegation](/images/route53-ns-records.png)

{{< note title="Note" >}}
It may take some time (minutes or hours) for the new DNS configuration to take effort.
{{< /note >}}

## Creating a Certificate

The Intake API endpoint in this tutorial is a public HTTPS endpoint and as such requires a valid SSL certificate.

The security resources playbook includes support for defining AWS Certificate Manager certificates, and in this section we will create a wildcard certificate for `*.demo.cloudhotspot.co`.

1\. Open the [security resources playbook you created earlier]({{< relref "security-resources/index.md" >}}).

2\. Add the highlighted configuration settings to the **demo-resources** environment, defined in `group_vars/demo-resources/vars.yml`:

{{< highlight python "hl_lines=9 10 11 12 13 14 15 16 17 18 19" >}}
# STS role in the target AWS account to assume
sts_role_arn: "arn:aws:iam::160775127577:role/admin"

# Trusted entities for the IAM admin role
# Here we trust the demo-users account
config_iam_admin_accounts:
  - 094411466117

# ACM certificates:
config_acm_certificates:
  "*.demo.cloudhotspot.co":
    SubjectAlternativeNames:
      - demo.cloudhotspot.co
    DomainValidationOptions:
      - DomainName: "*.demo.cloudhotspot.co"
        ValidationDomain: "cloudhotspot.co"
      - DomainName: "demo.cloudhotspot.co"
        ValidationDomain: "cloudhotspot.co"
{{< /highlight >}}

The optional `config_acm_certificates` variable defines a dictionary of named ACM certificates, and as you can see we add a single certificate that will have a subject name of `*.demo.cloudhotspot.co`.  

Notice that we configure a subject alternative name of the bare domain `demo.cloudhotspot.co`, and also configure a `ValidationDomain` of `cloudhotspot.co` for the subject name and subject alternative name domains of the certificate.

This validation domain can be either the domain of the subject name, or any parent domain of the subject name.  

In this example, we will be sending validation requests to the following email addresses:

- postmaster@cloudhotspot.co
- webmaster@cloudhotspot.co
- hostmaster@cloudhotspot.co
- administrator@cloudhotspot.co

3\. Run the security resources playbook targeting the `demo-resources` environment as demonstrated below:

{{< highlight bash >}}
$ export AWS_PROFILE=demo-resources-admin
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
{{< /highlight >}}

4\. The CloudFormation stack update will create a new AWS Certificate, which will generate a validation request to the emails we listed in the last step.  The stack update will wait for the validation process to either succeed or fail.

![Certificate Validation Email](/images/acm-email.png)

5\. After clicking the link in the validation email, the ACM approval page is presented, at which point you can click the **I Approve** button to approve the request.

![Certificate Validation Approval](/images/acm-approval.png)
![Certificate Validation Success](/images/acm-approval-success.png)

6\.  Note that because we are validating two domain names (`demo.cloudhotspot.co` and `*.demo.cloudhotspot.co`), you must approve separate validation requests for each domain.

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

4\.  Install the roles using the `ansible-galaxy` command as demonstrated below:

{{< highlight python >}}
$ ansible-galaxy install -r roles/requirements.yml --force
- extracting aws-cloudformation to /Users/jmenga/Source/casecommons/demo-proxy/roles/aws-cloudformation
- aws-cloudformation was installed successfully
- extracting aws-sts to /Users/jmenga/Source/casecommons/demo-proxy/roles/aws-sts
- aws-sts was installed successfully
{{< /highlight >}}

5\.  Modify the `group_vars/all/vars.yml` file, which contains global settings for the playbook:

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

7\. Remove the local `templates` folder, since we are using a template that is embedded within the `aws-cloudformation` role:

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
