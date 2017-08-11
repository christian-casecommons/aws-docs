---
date: 2017-03-21T01:37:17+13:00
title: Jenkins Stack
weight: 90
---

## Introduction

We have established all of the shared resource stacks in our **demo-users** and **demo-resources** accounts, created the Intake API application stack and Intake Accelerator application.

We will now deploy Jenkins application to our **demo-resources** account, creating a **demo** environment, with the Jenkins application published at `https://jenkins.demo.cloudhotspot.co`.

{{% note title="Note" %}}
The Jenkins stack relies on a number of supporting services and resources that we created in the [Intake API Stack section]({{< relref "intake-api/index.md" >}}) including:

- [Configuring DNS Delegation]({{< relref "intake-api/index.md#configuring-dns-delegation" >}})
- [Cloudformation Resources]({{< relref "cloudformation-resources/index.md" >}})
- [Creating a Certificate]({{< relref "intake-api/index.md#creating-a-certificate" >}})
- [Creating a ECR Repositories]({{< relref "ecr-resources/index.md#defining-a-new-environment" >}})
{{% /note %}}

## Creating the jenkins and jenkins-slave ECR Repositories

1\.  Add  nginx and nginx-slave as additional ecr repositories in demo-resources environment

Following demo for [EC2 Container Registry Resources]({{< relref "ecr-resources/index.md#introduction" >}}), we should now add two more repositories in our [demo-resources environment]({{< relref "ecr-resources/index.md#defining-a-new-environment" >}}) as follows.

```bash
$ cd demo-ecr-resources
$ tree -L 2
.
├── README.md
├── ansible.cfg
├── build
│   ├── 20170307112735
│   └── 20170322160313
├── group_vars
│   ├── all
│   ├── demo-resources
│   └── non-prod
├── inventory
├── roles
│   ├── aws-cloudformation
│   ├── aws-sts
│   └── requirements.yml
└── site.yml

10 directories, 5 files
```

2\.  Add jenkins and jenkins-slave as additional ecr repositories to `group_vars/demo-resources/vars.yml`:

{{< highlight python "hl_lines=12 13" >}}
# STS role to assume
sts_role_arn: arn:aws:iam::160775127577:role/admin

# Repositories
config_ecr_repos:
  casecommons/intake: {}
  casecommons/intake-api: {}
  casecommons/intake-base: {}
  casecommons/elasticsearch: {}
  casecommons/nginx: {}
  casecommons/squid: {}
  casecommons/jenkins: {}
  casecommons/jenkins-slave: {}
{{</highlight>}}

3\.  Running the Playbook

Now that we've updated environment settings for the **demo-resources** environment, let's run the playbook to create jenkins and jenkins-slave ECR repository resources for the environment.

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

3\.  The playbook will take a few minutes to create the CloudFormation stack and associated resources.  Whilst the CloudFormation stack is being created, you can review the CloudFormation stack that was generated in the `build/<timestamp>` folder:

4\. In the AWS console under **ECS > Repositories**, you should now be able to see your newly created repository in the **casecommons/jenkins** and **casecommons/jenkins-slave** repository:

![ECR Repositories](/images/ecr-repositories-with-jenkins.png)
## Publish Jenkins Master/Slave images

The Jenkins Docker image needs to be published to the **casecommons/jenkins** and **casecommons/jenkins-slave** ECR repository, so that it is available to ECS instances when we deploy the CloudFormation stack for running the Jenkins application.

1\. Clone the Jenkins application from the following forked repository (https://github.com/Casecommons/docker-jenkins) to your local environment

```bash
$ git clone git@github.com:Casecommons/docker-jenkins.git
Cloning into 'docker-jenkins'...
remote: Counting objects: 140, done.
remote: Compressing objects: 100% (56/56), done.
remote: Total 140 (delta 81), reused 134 (delta 75), pack-reused 0
Receiving objects: 100% (140/140), 25.46 KiB | 0 bytes/s, done.
Resolving deltas: 100% (81/81), done.
Checking connectivity... done.
$ cd docker-jenkins
```

2\. Open the `Makefile` at the root of the **docker-jenkins** repository and modify the highlighted settings:

{{< highlight make "hl_lines=3 4 5 6 9 10 13 14 15 16" >}}
# Project variables

DOCKER_REGISTRY ?= 429614120872.dkr.ecr.us-west-2.amazonaws.com
ORG_NAME ?= cwds
REPO_NAME ?= jenkins
SLAVE_REPO_NAME ?= jenkins-slave

# AWS settings
AWS_ROLE ?= remoteAdmin
KMS_KEY_ID ?= 3ea941bf-ee54-4941-8f77-f1dd417667cd

# Jenkins settings
export DOCKER_GID ?= 100
export JENKINS_USERNAME ?= admin
export JENKINS_PASSWORD ?= password
export KMS_JENKINS_PASSWORD ?=
AQECAHgohc0dbuzR1L3lEdEkDC96PMYUEV9nITogJU2vbocgQAAAAGowaAYJKoZIhvcNAQcGoFswWQIBADBUBgkqhkiG9w0BBwEwHgYJYIZIAWUDBAEuMBEEDEDRPvtPNmH1mcmplwIBEIAn4V2lcp2J8/4SYtKrE2deH15dkNCKiIun0biaev+Dwv3K1KOnRTyU
export JENKINS_SLAVE_VERSION ?= 2.2
export JENKINS_SLAVE_LABELS ?= DOCKER
...
...
{{< /highlight >}}

Here we ensure the `Makefile` settings are configured with the correct Docker registry name (`DOCKER_REGISTRY`), organization name (`ORG_NAME`), repository name (`REPO_NAME`), AWS assume role (`AWS_ROLE`), KMS Master Key (`KMS_KEY_ID`), Docker group id (`DOCKER_GID`), Jenkins user name (`JENKINS_USERNAME`), Jenkins user password (`JENKINS_PASSWORD`), KMS ciphertext Jenkins password (`KMS_JENKINS_PASSWORD`).

3\. Run `make secret my-secret-password` command to generate KMS_JENKINS_PASSWORD

```bash
$ make secret my-secret-password
=> Encrypted ciphertext:
AQECAHipA5t7CpNKiZ5vP0gUjbCZImphz7KQJsH+JQoh9RzzxAAAAHAwbgYJKoZIhvcNAQcGoGEwXwIBADBaBgkqhkiG9w0BBwEwHgYJYIZIAWUDBAEuMBEEDKTX2LYxLRDbooQxdQIBEIAtuu4ooTAnMnEGLDACCSw3clzMZD3SDLtWTNBbp4Jsp8pp842goCPCNxJC0Dsc
```

4\. Run the `make login` command to login to the ECR repository for the **demo-resources** account:

```bash
$ export AWS_PROFILE=demo-resources-admin
$ make login
=> Logging in to Docker registry ...
Enter MFA code: *****
Login Succeeded
=> Logged in to Docker registry
```

5\. Run the `make build` command, which creates docker jenkins images:

```bash
$ make build
make build
=> Building image...
Step 1/15 : FROM jenkins:2.32.3-alpine
2.32.3-alpine: Pulling from library/jenkins
627beaf3eaaf: Pull complete
1de20f2d8b83: Pull complete
3e00029ebfe3: Pull complete
1839d7896efa: Pull complete
750cfd608edc: Pull complete
235d8aedfa97: Pull complete
90a93da4c692: Pull complete
be93cb336ed7: Pull complete
2627b3478cf4: Pull complete
68e0a918969e: Pull complete
f5f0989d3476: Pull complete
9ebff3be068f: Pull complete
79d80188d4e1: Pull complete
e2e367ea4282: Pull complete
Digest: sha256:905b236c83bef3bfddf4ab5fbe32f49bdd1362f96d3aad5a5ba1e32b431d08f4
Status: Downloaded newer image for jenkins:2.32.3-alpine
....
....
Removing intermediate container b6a60b5da7d2
Successfully built fa22b4dc60f9
=> Build complete
```

6\. Run the `make publish` command, which will tag images and publish images to docker repository:

```bash
$ make publish
=> Tagging Jenkins images with tags latest 20170323101229.08e79eb 08e79eb
=> Publishing Jenkins images...
...
...
latest: digest:
sha256:92a6977e1eadbf00962f93fc32aec4a9e35cfcc63b1354a587e5fb2305091795 size:
2202
=> Publish complete
```


7\. Run the `make clean` command to clean up the local Docker environment.

```bash
$ make clean
=> Stopping services...
...
Removing dockerjenkins_jenkins-slave_1 ... done
Removing dockerjenkins_jenkins_1 ... done
Removing network dockerjenkins_default
WARNING: Network dockerjenkins_default not found.
Volume jenkins_home is external, skipping
=> Removing dangling images...
=> Cleanup complete
```

8\. In the AWS console under **ECS > Repositories**, you should now be able to see your newly published image in the **casecommons/jenkins** and **casecommons/jenkins-slave** repository:

![Jenkins docker image](/images/ecr-repisitory-for-jenkins.png)

## Installing the Playbook

We now have all the supporting pieces in place to deploy the Jenkins stack to the **demo resources** account.  Instead of creating a new playbook as we have done previously in this tutorial, we will instead clone an existing playbook and add a new environment called **demo-resources** to the playbook.

1\. Clone the [Jenkins AWS project](https://github.com/Casecommons/jenkins-aws) to your local environment.

```bash
$ git clone git@github.com:Casecommons/jenkins-aws.git
Cloning into 'jenkins-aws'...
remote: Counting objects: 46, done.
remote: Compressing objects: 100% (23/23), done.
remote: Total 46 (delta 14), reused 43 (delta 11), pack-reused 0
Receiving objects: 100% (46/46), 10.40 KiB | 0 bytes/s, done.
Resolving deltas: 100% (14/14), done.
Checking connectivity... done.
$ cd jenkins-aws
```

3\.  Install the required Ansible roles for the playbook using the `ansible-galaxy` command as demonstrated below:

{{< highlight python >}}
$ ansible-galaxy install -r roles/requirements.yml --force
- extracting aws-cloudformation to /Users/pemageyleg/Work/aws/demo/jenkins-aws/roles/aws-cloudformation
- aws-cloudformation was installed successfully
- extracting aws-sts to /Users/pemageyleg/Work/aws/demo/jenkins-aws/roles/aws-sts
- aws-sts was installed successfully
{{< /highlight >}}

5\.  Review the `group_vars/all/vars.yml` file, which contains global settings for the playbook:

{{< highlight python >}}
cf_stack_name: "jenkins-{{ env }}"
cf_stack_tags:
  org:business:owner: CA Intake
  org:business:product: Jenkins Server
  org:business:severity: High
  org:tech:environment: "{{ env }}"
  org:tech:contact: pema@casecommons.org
{{< /highlight >}}

You can see this stack has many different stack inputs, which revolve around application settings.

One thing to note is that the `cf_stack_template` variable is not defined - this variable defaults to the path `templates/stack.yml.j2` in the local playbook, and you will find this playbook includes a large CloudFormation template in this location.

## Defining a New Environment

We will now add a new environment called **demo** to the playbook, which will be used to create the Intake API application stack in the **demo-resources** account.

1\. Modify the `inventory` file so that it defines a new environment called **demo**, ensuring you specify `ansible_connection=local`:

{{< highlight ini "hl_lines=4 5" >}}
[dev]
dev ansible_connection=local

[demo]
demo ansible_connection=local
```
{{< /highlight >}}

2\. Create a file called `group_vars/demo/vars.yml`, which will hold all environment specific configuration for the **demo** environment:

```bash
$ mkdir -p group_vars/demo
$ touch group_vars/demo/vars.yml
```

4\. Copy the existing settings from `group_vars/non-prod/vars.yml` to `group_vars/demo/vars.yml` and modify as shown below:

{{< highlight ini "hl_lines=2 5 8 9 10 11 14 15 16 17 21 26" >}}
# STS role settings
sts_role_arn: "arn:aws:iam::429614120872:role/remoteAdmin"

# Target VPC
config_vpc_name: "Default"

# Application settings
config_application_image_id: ami-bcdc54dc
config_application_key_name: admin-shared-key
config_application_docker_image: 429614120872.dkr.ecr.us-west-2.amazonaws.com/cwds/jenkins
config_application_domain: ca.mycasebook.org

# Worker settings
config_worker_instance_type: t2.medium
config_worker_image_id: ami-bcdc54dc
config_worker_key_name: admin-shared-key
config_worker_docker_image: 429614120872.dkr.ecr.us-west-2.amazonaws.com/cwds/jenkins-slave
config_worker_service_desired_count: 1

# Jenkins settings
config_jenkins_password: AQECAHgohc0dbuzR1L3lEdEkDC96PMYUEV9nITogJU2vbocgQAAAAG4wbAYJKoZIhvcNAQcGoF8wXQIBADBYBgkqhkiG9w0BBwEwHgYJYIZIAWUDBAEuMBEEDG+ksTd04UBiWQgM8gIBEIAr8R1/ZhsZEUpEVKGiqwsiW5PTnhmuxeLWjObS7fkw6xfeQwKB6r4L9gSpwg==
config_jenkins_slave_initial_heap_size: 2g
config_jenkins_slave_max_heap_size: 4g

# Load balancer settings
config_lb_certificate_arn:
  Fn::ImportValue: CaMycasebookOrgCertArn
{{< /highlight >}}

Here we target the **demo-resources** account by specifying the **demo-resources** IAM **admin** role in the `sts_role_arn` variable, whilst the remaining settings configure the Jenkins stack specify to the **demo-resources** template account:

- `config_application_image_id` - specifies the AMI ID of the image used to create the ECS container instances.  Notice this matches the ID of the [AMI created earlier]({{< relref "web-proxy/index.md#creating-an-ecs-ami" >}})
- `config_application_key_name` - specifies the name of the EC2 key pair that ECS container instances will be created with.  Notice this matches the name of the [EC2 key pair created earlier]({{< relref "web-proxy/index.md#creating-an-ec2-key-pair" >}}).
- `config_application_docker_image` - specifies the Docker image used to run the Jenkins master containers.  Notice this matches the [image we created earlier]({{< relref "jenkins/index.md#creating-the-jenkins-and-jenkins-slave-ecr-repositories" >}})
- `config_application_domain` - specifies the base domain that our application will be served from.
config_worker_image_id: specifies the AMI ID of the image used to create the ECS container instances.  Notice this matches the ID of the [AMI created earlier]({{< relref "web-proxy/index.md#creating-an-ecs-ami" >}})
config_worker_key_name: specifies the name of the EC2 key pair that ECS container instances will be created with.  Notice this matches the name of the [EC2 key pair created earlier]({{< relref "web-proxy/index.md#creating-an-ec2-key-pair" >}}).
-d-key
config_worker_docker_image: specifies the Docker image used to run the Jenkins master containers.  Notice this matches the [image we created earlier]({{< relref "jenkins/index.md#creating-the-jenkins-and-jenkins-slave-ecr-repositories" >}})
- config_jenkins_password: encrypted ciphertext password for jenkins after using kms AQECAHgohc0dbuzR1L3lEdEkDC96PMYUEV9nITogJU2vbocgQAAAAG4wbAYJKoZIhvcNAQcGoF8wXQIBADBYBgkqhkiG9w0BBwEwHgYJYIZIAWUDBAEuMBEEDG+ksTd04UBiWQgM8gIBEIAr8R1/ZhsZEUpEVKGiqwsiW5PTnhmuxeLWjObS7fkw6xfeQwKB6r4L9gSpwg==
- `config_lb_certificate_arn` - specifies the ARN of the AWS certificate manager (ACM) certificate to serve from the application load balancer for HTTPS connections.  Notice that we can specify this setting as a CloudFormation instrinsic function, as the template is configured to cast the configured value to a JSON object.  The intrinsic function imports the CloudFormation export `DemoCloudhotspotCoCertificateArn`, which was created earlier when we [created the certificate]({{< relref "intake-api/index.md#creating-a-certificate" >}}) in the security resources playbook.


2\. In a local shell configured with an admin profile targetting the **demo resources** account, run the following command to generate ciphertext for the `config_jenkins_password` variable:

```bash
$ export AWS_PROFILE=demo-resources-admin
$ aws kms encrypt --key-id 11710b57-c424-4df7-ab3d-20358820edd9 --plaintext $(openssl rand -base64 32)
Enter MFA code: ******
'{
    "KeyId": "arn:aws:kms:us-west-2:160775127577:key/11710b57-c424-4df7-ab3d-20358820edd9",
    "CiphertextBlob": "AQECAHjbSbOZ8FLk7XffvdtrDewDyQKH9bOaMrY6jf+N3si+SQAAAIswgYgGCSqGSIb3DQEHBqB7MHkCAQAwdAYJKoZIhvcNAQcBMB4GCWCGSAFlAwQBLjARBAxUVkZgkHNkRWRcbtgCARCAR2bG8d33uID+nq01bRjHeNJzsFgTqrOxCoY7LmR+tyVT/3oTAQYePtsJC3Dt9zmOJ4G82Q36X4q0ng5r8YPaj15wp/en0tPY"
}
```

3\. Copy the `CiphertextBlob` value from the `aws kms encrypt` command output and set the `config_jenkins_password` variable in `group_vars/demo/vars.yml` to this value:

## Running the Playbook

Now that we've defined environment settings for the **demo** environment targeting our **demo-resources** account, let's run the playbook.

1\. Ensure your local AWS environment is configured to target the **demo-resources** account:

```bash
$ export AWS_PROFILE=demo-resources-admin
```

2\. Run the Ansible playbook targeting the `demo` environment as demonstrated below:

{{< highlight bash >}}
$ ansible-playbook site.yml -e env=demo

PLAY [Assume Role] *************************************************************

TASK [aws-sts : set_fact] ******************************************************
ok: [demo]

TASK [aws-sts : checking if sts functions are sts_disabled] ********************
skipping: [demo]

TASK [aws-sts : setting empty sts_session_output result] ***********************
skipping: [demo]

TASK [aws-sts : setting sts_creds if legacy AWS credentials are present (e.g. for Ansible Tower)] ***
skipping: [demo]

TASK [aws-sts : assume sts role] ***********************************************
Enter MFA code: ******
ok: [demo]
...
...
TASK [aws-cloudformation : set local path fact if s3 upload disabled] **********
ok: [demo]

TASK [aws-cloudformation : configure application stack] ************************
...
...
PLAY RECAP *********************************************************************
demo                       : ok=18   changed=1    unreachable=0    failed=0
{{< /highlight >}}

3\.  The playbook will take approxiately 10 minutes to create the CloudFormation stack and associated resources.  Whilst the CloudFormation stack is being created, you can review the CloudFormation stack that was generated in the `build/<timestamp>` folder:

```bash
$ tree build
build
└── 20170301224832
    ├── jenkins-demo-config.json
    ├── jenkins-demo-policy.json
    ├── jenkins-demo-stack.json
    └── jenkins-demo-stack.yml
```

The following shows the `jenkins-demo-stack.yml` file that was generated and uploaded to CloudFormation:

```python
AWSTemplateFormatVersion: "2010-09-09"

Description: Jenkins - demo-resources

Conditions:
  WorkerSingleInstanceCondition:
    Fn::Equals:
      - Ref: WorkerServiceDesiredCount
      - 1

Parameters:
  ApplicationImageId:
    Type: String
    Description: Application Amazon Machine Image Id
  ApplicationInstanceType:
    Type: String
    Description: Application EC2 Instance Type
    Default: t2.micro
    AllowedValues:
      - t2.micro
      - t2.small
      - t2.medium
      - t2.large
      - m4.large
  ApplicationAutoscalingDesiredCount:
    Type: Number
    Description: Application AutoScaling Group Desired Count
    Default: 1
  ApplicationDockerImage:
    Type: String
    Description: Docker Image for Application
  ApplicationDockerImageTag:
    Type: String
    Description: Docker Image Tag for Application
    Default: latest
  ApplicationKeyName:
    Type: String
    Description: EC2 Key Pair for Application SSH Access
  ApplicationDomain:
    Type: String
    Description: Base public domain of the application URL
  ApplicationFileSystemMount:
    Type: String
    Description: EFS file system mount path
    Default: /mnt/efs/jenkins
  DockerGroupId:
    Type: Number
    Description: Docker Group ID of ECS Container Instance
    Default: 497
  WorkerImageId:
    Type: String
    Description: Worker Amazon Machine Image Id
  WorkerInstanceType:
    Type: String
    Description: Worker EC2 Instance Type
    Default: t2.micro
    AllowedValues:
      - t2.micro
      - t2.small
      - t2.medium
      - t2.large
      - m4.large
  WorkerAutoscalingDesiredCount:
    Type: Number
    Description: Worker AutoScaling Group Desired Count
    Default: 1
  WorkerServiceDesiredCount:
    Type: Number
    Description: Worker Service Desired Count
    Default: 1
  WorkerDockerImage:
    Type: String
    Description: Docker Image for Worker
  WorkerDockerImageTag:
    Type: String
    Description: Docker Image Tag for Worker
    Default: latest
  WorkerKeyName:
    Type: String
    Description: EC2 Key Pair for Worker SSH Access
  LogRetention:
    Type: Number
    Description: Log retention in days
    Default: 7
    AllowedValues: [1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365, 400, 545, 731, 1827, 3653]
  JenkinsMasterPort:
    Type: Number
    Description: Jenkins Master HTTP Port
    Default: 8080
  JenkinsMasterInitialHeapSize:
    Type: String
    Description: JVM Initial Heap Size for Jenkins Master
    Default: "256m"
  JenkinsMasterMaxHeapSize:
    Type: String
    Description: JVM Max Heap Size for Jenkins Master
    Default: "1g"
  JenkinsSlaveInitialHeapSize:
    Type: String
    Description: JVM Initial Heap Size for Jenkins Slaves
    Default: "512m"
  JenkinsSlaveMaxHeapSize:
    Type: String
    Description: JVM Max Heap Size for Jenkins Slaves
    Default: "1g"
  JenkinsSlavePort:
    Type: Number
    Description: Jenkins Agent JNLP Port
    Default: 50000
  JenkinsUsername:
    Type: String
    Description: Jenkins Administrative User
    Default: admin
  JenkinsPassword:
    Type: String
    Description: KMS encrypted Jenkins password
  JenkinsUserId:
    Type: String
    Description: Jenkins User Id that owns the EFS File System Mount
    Default: "1000"

Resources:
  ApplicationDnsRecord:
    Type: "AWS::Route53::RecordSet"
    Properties:
      Name:
        Fn::Sub: "jenkins.${ApplicationDomain}"
      TTL: "300"
      HostedZoneName:
        Fn::Sub: "${ApplicationDomain}."
      Type: "CNAME"
      Comment:
        Fn::Sub: "${AWS::StackName} Application Record"
      ResourceRecords:
        - Fn::Sub: "${ApplicationLoadBalancer.DNSName}"
  ApplicationLoadBalancer:
    Type: "AWS::ElasticLoadBalancing::LoadBalancer"
    Properties:
      Scheme: "internet-facing"
      SecurityGroups:
        - Ref: "ApplicationLoadBalancerSecurityGroup"
      Subnets:
        - Fn::ImportValue: "DefaultPublicSubnetA"
        - Fn::ImportValue: "DefaultPublicSubnetB"
      CrossZone: "true"
      ConnectionSettings:
        IdleTimeout: 600
      ConnectionDrainingPolicy:
        Enabled: "true"
        Timeout: 60
      Listeners:
        - LoadBalancerPort: "443"
          InstancePort: { "Ref": "JenkinsMasterPort" }
          Protocol: "https"
          SSLCertificateId: {"Fn::ImportValue": "DemoPemageylegComCertificateArn"}
        - LoadBalancerPort: { "Ref": "JenkinsSlavePort" }
          InstancePort: { "Ref": "JenkinsSlavePort" }
          Protocol: "tcp"
      HealthCheck:
        Target:
          Fn::Sub: "HTTP:${JenkinsMasterPort}/login"
        HealthyThreshold: "2"
        UnhealthyThreshold: "10"
        Interval: "30"
        Timeout: "5"
      Tags:
        - Key: "Name"
          Value:
            Fn::Sub: "${AWS::StackName}-ApplicationLoadBalancer"
  ApplicationLoadBalancerSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      GroupDescription: "Application Load Balancer Security Group"
      VpcId:
        Fn::ImportValue: "DefaultVpcId"
      SecurityGroupIngress:
        - IpProtocol: "tcp"
          FromPort: "443"
          ToPort: "443"
          CidrIp: "0.0.0.0/0"
        - IpProtocol: "tcp"
          FromPort: { "Ref": "JenkinsSlavePort" }
          ToPort: { "Ref": "JenkinsSlavePort" }
          CidrIp: "0.0.0.0/0"
  ApplicationLoadBalancerToApplicationAutoscalingEgress:
    Type: "AWS::EC2::SecurityGroupEgress"
    Properties:
      IpProtocol: "tcp"
      FromPort: { "Ref": "JenkinsMasterPort" }
      ToPort: { "Ref": "JenkinsMasterPort" }
      GroupId: { "Ref": "ApplicationLoadBalancerSecurityGroup" }
      DestinationSecurityGroupId: { "Ref": "ApplicationAutoscalingSecurityGroup" }
  ApplicationLoadBalancerToApplicationAutoscalingIngress:
    Type: "AWS::EC2::SecurityGroupIngress"
    Properties:
      IpProtocol: "tcp"
      FromPort: { "Ref": "JenkinsMasterPort" }
      ToPort: { "Ref": "JenkinsMasterPort" }
      GroupId: { "Ref": "ApplicationAutoscalingSecurityGroup" }
      SourceSecurityGroupId: { "Ref": "ApplicationLoadBalancerSecurityGroup" }
  ApplicationLoadBalancerToApplicationAutoscalingSlaveEgress:
    Type: "AWS::EC2::SecurityGroupEgress"
    Properties:
      IpProtocol: "tcp"
      FromPort: { "Ref": "JenkinsSlavePort" }
      ToPort: { "Ref": "JenkinsSlavePort" }
      GroupId: { "Ref": "ApplicationLoadBalancerSecurityGroup" }
      DestinationSecurityGroupId: { "Ref": "ApplicationAutoscalingSecurityGroup" }
  ApplicationLoadBalancerToApplicationAutoscalingSlaveIngress:
    Type: "AWS::EC2::SecurityGroupIngress"
    Properties:
      IpProtocol: "tcp"
      FromPort: { "Ref": "JenkinsSlavePort" }
      ToPort: { "Ref": "JenkinsSlavePort" }
      GroupId: { "Ref": "ApplicationAutoscalingSecurityGroup" }
      SourceSecurityGroupId: { "Ref": "ApplicationLoadBalancerSecurityGroup" }
  ApplicationAutoscalingLaunchConfiguration:
    Type: "AWS::AutoScaling::LaunchConfiguration"
    Metadata:
      AWS::CloudFormation::Init:
        config:
          commands:
            10_mount_create:
              command:
                Fn::Sub: "sudo mkdir -p ${ApplicationFileSystemMount}"
            11_mount_fstab:
              command:
                Fn::Sub: echo -e "${ApplicationFileSystem}.efs.${AWS::Region}.amazonaws.com:/ \t\t ${ApplicationFileSystemMount} \t\t nfs4 \t\t defaults \t\t 0 \t\t 0" | sudo tee -a /etc/fstab
            12_mount_efs:
              command:
                Fn::Sub: "sudo mount -a"
            13_mount_permissions:
              command:
                Fn::Sub: "sudo chown ${JenkinsUserId}:${JenkinsUserId} ${ApplicationFileSystemMount}"
            20_first_run:
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
      KeyName: { "Ref": "ApplicationKeyName" }
      SecurityGroups:
        - Ref: ApplicationAutoscalingSecurityGroup
      UserData:
        Fn::Base64:
          Fn::Join: ["", [
            "#!/bin/bash\n",
            "/opt/aws/bin/cfn-init -v ",
            "    --stack ", { "Ref" : "AWS::StackName" },
            "    --resource ApplicationAutoscalingLaunchConfiguration ",
            "    --region ", { "Ref" : "AWS::Region" },
            "\n",
            "/opt/aws/bin/cfn-signal -e $? --stack ", { "Ref" : "AWS::StackName" },
            "    --resource ApplicationAutoscaling ",
            "    --region ", { "Ref" : "AWS::Region" },
            "\n",
          ] ]
  ApplicationAutoscaling:
    Type: "AWS::AutoScaling::AutoScalingGroup"
    DependsOn:
      - ApplicationMountTargetA
      - ApplicationMountTargetB
      - ApplicationDmesgLogGroup
      - ApplicationDockerLogGroup
      - ApplicationEcsAgentLogGroup
      - ApplicationEcsInitLogGroup
      - ApplicationMessagesLogGroup
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: { "Ref": "ApplicationAutoscalingDesiredCount" }
        MinSuccessfulInstancesPercent: "100"
        WaitOnResourceSignals: "true"
        PauseTime: "PT15M"
    CreationPolicy:
      ResourceSignal:
        Count: { "Ref": "ApplicationAutoscalingDesiredCount" }
        Timeout: "PT15M"
    Properties:
      VPCZoneIdentifier:
        - Fn::ImportValue: "DefaultPublicSubnetA"
        - Fn::ImportValue: "DefaultPublicSubnetB"
      LaunchConfigurationName: { "Ref" : "ApplicationAutoscalingLaunchConfiguration" }
      MinSize: "0"
      MaxSize: "4"
      DesiredCapacity: { "Ref": "ApplicationAutoscalingDesiredCount" }
      LoadBalancerNames: []
      Tags:
        - Key: "Name"
          Value:
            Fn::Sub: "${AWS::StackName}-ApplicationAutoscaling-instance"
          PropagateAtLaunch: "true"
  ApplicationAutoscalingSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      GroupDescription: "Application Autoscaling Security Group"
      VpcId:
        Fn::ImportValue: "DefaultVpcId"
      SecurityGroupIngress:
        - IpProtocol: "tcp"
          FromPort: "22"
          ToPort: "22"
          CidrIp:
            Fn::ImportValue: "DefaultVpcCidr"
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
        - IpProtocol: "tcp"
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: "udp"
          FromPort: 123
          ToPort: 123
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: "Name"
          Value:
            Fn::Sub: "${AWS::StackName}-ApplicationAutoscalingSecurityGroup"
  ApplicationAutoscalingToApplicationFileSystemEgress:
    Type: "AWS::EC2::SecurityGroupEgress"
    Properties:
      IpProtocol: "tcp"
      FromPort: 2049
      ToPort: 2049
      GroupId: { "Ref": "ApplicationAutoscalingSecurityGroup" }
      DestinationSecurityGroupId: { "Ref": "ApplicationFileSystemSecurityGroup" }
  ApplicationAutoscalingToApplicationFileSystemIngress:
    Type: "AWS::EC2::SecurityGroupIngress"
    Properties:
      IpProtocol: "tcp"
      FromPort: 2049
      ToPort: 2049
      GroupId: { "Ref": "ApplicationFileSystemSecurityGroup" }
      SourceSecurityGroupId: { "Ref": "ApplicationAutoscalingSecurityGroup" }
  ApplicationAutoscalingInstanceRole:
    Type: "AWS::IAM::Role"
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: "Allow"
            Principal:
              Service: [ "ec2.amazonaws.com" ]
            Action: [ "sts:AssumeRole" ]
      Policies:
        - PolicyName: "EC2ContainerInstancePolicy"
          PolicyDocument:
            Version: "2012-10-17"
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
              - Effect: "Allow"
                Action:
                - "kms:Decrypt"
                - "kms:DescribeKey"
                Resource:
                  Fn::ImportValue: "CfnMasterKeyArn"
        - PolicyName: "CloudwatchLogsPolicy"
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: "Allow"
                Action:
                - "logs:CreateLogStream"
                - "logs:PutLogEvents"
                - "logs:DescribeLogStreams"
                Resource:
                  Fn::Sub: "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:${AWS::StackName}*"
  ApplicationAutoscalingInstanceProfile:
    Type: "AWS::IAM::InstanceProfile"
    Properties:
      Roles:
        - Ref: "ApplicationAutoscalingInstanceRole"
  ApplicationFileSystem:
    Type: "AWS::EFS::FileSystem"
    Properties:
      FileSystemTags:
        - Key: Name
          Value:
            Fn::Sub: ${AWS::StackName}-application-filesystem
  ApplicationMountTargetA:
    Type: "AWS::EFS::MountTarget"
    Properties:
      FileSystemId: { "Ref": "ApplicationFileSystem" }
      SecurityGroups:
        - Ref: "ApplicationFileSystemSecurityGroup"
      SubnetId:
        Fn::ImportValue: DefaultHighSubnetA
  ApplicationMountTargetB:
    Type: "AWS::EFS::MountTarget"
    Properties:
      FileSystemId: { "Ref": "ApplicationFileSystem" }
      SecurityGroups:
        - Ref: "ApplicationFileSystemSecurityGroup"
      SubnetId:
        Fn::ImportValue: DefaultHighSubnetB
  ApplicationFileSystemSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      GroupDescription: "Application EFS Security Group"
      VpcId:
        Fn::ImportValue: "DefaultVpcId"
      SecurityGroupIngress: []
      SecurityGroupEgress:
        - IpProtocol: "icmp"
          FromPort: -1
          ToPort: -1
          CidrIp: 192.0.2.0/32
  WorkerAutoscalingLaunchConfiguration:
    Type: "AWS::AutoScaling::LaunchConfiguration"
    Metadata:
      AWS::CloudFormation::Init:
        config:
          commands:
            10_first_run:
              command: "sh firstrun.sh"
              env:
                STACK_NAME: { "Ref": "AWS::StackName" }
                AUTOSCALING_GROUP: "WorkerAutoscaling"
                AWS_DEFAULT_REGION: { "Ref": "AWS::Region" }
                ECS_CLUSTER: { "Ref": "WorkerCluster" }
              cwd: "/home/ec2-user/"
          files:
            /etc/ecs/ecs.config:
              content:
                Fn::Sub: "ECS_CLUSTER=${WorkerCluster}\n"
    Properties:
      ImageId: { "Ref": "WorkerImageId" }
      InstanceType: { "Ref": "WorkerInstanceType" }
      AssociatePublicIpAddress: "true"
      IamInstanceProfile: { "Ref": "WorkerAutoscalingInstanceProfile" }
      KeyName: { "Ref": "WorkerKeyName" }
      SecurityGroups:
        - Ref: WorkerAutoscalingSecurityGroup
      UserData:
        Fn::Base64:
          Fn::Join: ["", [
            "#!/bin/bash\n",
            "/opt/aws/bin/cfn-init -v ",
            "    --stack ", { "Ref" : "AWS::StackName" },
            "    --resource WorkerAutoscalingLaunchConfiguration ",
            "    --region ", { "Ref" : "AWS::Region" },
            "\n",
            "/opt/aws/bin/cfn-signal -e $? --stack ", { "Ref" : "AWS::StackName" },
            "    --resource WorkerAutoscaling ",
            "    --region ", { "Ref" : "AWS::Region" },
            "\n",
          ] ]
  WorkerAutoscaling:
    Type: "AWS::AutoScaling::AutoScalingGroup"
    DependsOn:
      - WorkerDmesgLogGroup
      - WorkerDockerLogGroup
      - WorkerEcsAgentLogGroup
      - WorkerEcsInitLogGroup
      - WorkerMessagesLogGroup
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: { "Ref": "WorkerAutoscalingDesiredCount" }
        MinSuccessfulInstancesPercent: "100"
        WaitOnResourceSignals: "true"
        PauseTime: "PT15M"
    CreationPolicy:
      ResourceSignal:
        Count: { "Ref": "WorkerAutoscalingDesiredCount" }
        Timeout: "PT15M"
    Properties:
      VPCZoneIdentifier:
        - Fn::ImportValue: "DefaultPublicSubnetA"
        - Fn::ImportValue: "DefaultPublicSubnetB"
      LaunchConfigurationName: { "Ref" : "WorkerAutoscalingLaunchConfiguration" }
      MinSize: "0"
      MaxSize: "4"
      DesiredCapacity: { "Ref": "WorkerAutoscalingDesiredCount" }
      LoadBalancerNames: []
      Tags:
        - Key: "Name"
          Value:
            Fn::Sub: "${AWS::StackName}-WorkerAutoscaling-instance"
          PropagateAtLaunch: "true"
  WorkerAutoscalingSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      GroupDescription: "Worker Autoscaling Security Group"
      VpcId:
        Fn::ImportValue: "DefaultVpcId"
      SecurityGroupIngress:
        - IpProtocol: "tcp"
          FromPort: "22"
          ToPort: "22"
          CidrIp:
            Fn::ImportValue: "DefaultVpcCidr"
      SecurityGroupEgress:
        - IpProtocol: "tcp"
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: "tcp"
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
        - IpProtocol: "tcp"
          FromPort: 50000
          ToPort: 50000
          CidrIp: 0.0.0.0/0
        - IpProtocol: "udp"
          FromPort: 53
          ToPort: 53
          CidrIp:
            Fn::Join: ["", [
              "Fn::ImportValue": "DefaultVpcDnsServer", "/32"
            ] ]
        - IpProtocol: "tcp"
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: "udp"
          FromPort: 123
          ToPort: 123
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: "Name"
          Value:
            Fn::Sub: "${AWS::StackName}-WorkerAutoscalingSecurityGroup"
  WorkerAutoscalingToApplicationLoadBalancerEgress:
    Type: "AWS::EC2::SecurityGroupEgress"
    Properties:
      IpProtocol: "tcp"
      FromPort: { "Ref": "JenkinsSlavePort" }
      ToPort:  { "Ref": "JenkinsSlavePort" }
      GroupId: { "Ref": "WorkerAutoscalingSecurityGroup" }
      DestinationSecurityGroupId: { "Ref": "ApplicationLoadBalancerSecurityGroup" }
  WorkerAutoscalingToApplicationLoadBalancerIngress:
    Type: "AWS::EC2::SecurityGroupIngress"
    Properties:
      IpProtocol: "tcp"
      FromPort: { "Ref": "JenkinsSlavePort" }
      ToPort:  { "Ref": "JenkinsSlavePort" }
      GroupId: { "Ref": "ApplicationLoadBalancerSecurityGroup" }
      SourceSecurityGroupId: { "Ref": "WorkerAutoscalingSecurityGroup" }
  WorkerAutoscalingInstanceRole:
    Type: "AWS::IAM::Role"
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: "Allow"
            Principal:
              Service: [ "ec2.amazonaws.com" ]
            Action: [ "sts:AssumeRole" ]
      ManagedPolicyArns:
        - "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser"
      Policies:
        - PolicyName: "EC2ContainerInstancePolicy"
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: "Allow"
                Action:
                  - "ecs:RegisterContainerInstance"
                  - "ecs:DeregisterContainerInstance"
                Resource:
                  Fn::Sub: "arn:aws:ecs:${AWS::Region}:${AWS::AccountId}:cluster/${WorkerCluster}"
              - Effect: "Allow"
                Action:
                  - "ecs:DiscoverPollEndpoint"
                  - "ecs:Submit*"
                  - "ecs:Poll"
                  - "ecs:StartTelemetrySession"
                Resource: "*"
              - Effect: "Allow"
                Action:
                - "kms:Decrypt"
                - "kms:DescribeKey"
                Resource:
                  Fn::ImportValue: "CfnMasterKeyArn"
        - PolicyName: "CloudwatchLogsPolicy"
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: "Allow"
                Action:
                - "logs:CreateLogStream"
                - "logs:PutLogEvents"
                - "logs:DescribeLogStreams"
                Resource:
                  Fn::Sub: "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:${AWS::StackName}*"
  WorkerAutoscalingInstanceProfile:
    Type: "AWS::IAM::InstanceProfile"
    Properties:
      Roles:
        - Ref: "WorkerAutoscalingInstanceRole"
  ApplicationCluster:
    Type: "AWS::ECS::Cluster"
  WorkerCluster:
    Type: "AWS::ECS::Cluster"
  JenkinsTaskDefinition:
    Type: "AWS::ECS::TaskDefinition"
    Properties:
      NetworkMode: host
      Volumes:
      - Name: "jenkins-home"
        Host:
          SourcePath: { "Ref": "ApplicationFileSystemMount" }
      - Name: "docker"
        Host:
          SourcePath: "/var/run/docker.sock"
      ContainerDefinitions:
      - Name: "jenkins"
        Image:
          Fn::Sub: ${ApplicationDockerImage}:${ApplicationDockerImageTag}
        MemoryReservation: 100
        PortMappings:
          - ContainerPort: { "Ref": "JenkinsMasterPort" }
        MountPoints:
          - SourceVolume: "jenkins-home"
            ContainerPath: "/var/jenkins_home"
          - SourceVolume: "docker"
            ContainerPath: "/var/run/docker.sock"
        LogConfiguration:
          LogDriver: awslogs
          Options:
            awslogs-group:
              Fn::Sub: ${AWS::StackName}/ecs/JenkinsService/jenkins
            awslogs-region: { "Ref": "AWS::Region" }
            awslogs-stream-prefix: docker
        Environment:
          - Name: "JAVA_TOOL_OPTIONS"
            Value:
              Fn::Sub: "-Xmx${JenkinsMasterMaxHeapSize} -Xms${JenkinsMasterInitialHeapSize}"
          - Name: "DOCKER_GID"
            Value: { "Ref": "DockerGroupId" }
          - Name: "JENKINS_USERNAME"
            Value: { "Ref": "JenkinsUsername" }
          - Name: "KMS_JENKINS_PASSWORD"
            Value: { "Ref": "JenkinsPassword" }
  WorkerTaskDefinition:
    Type: "AWS::ECS::TaskDefinition"
    Properties:
      Volumes:
      - Name: "docker"
        Host:
          SourcePath: "/var/run/docker.sock"
      ContainerDefinitions:
      - Name: "slave"
        Image:
          Fn::Sub: ${WorkerDockerImage}:${WorkerDockerImageTag}
        MemoryReservation: 100
        MountPoints:
          - SourceVolume: "docker"
            ContainerPath: "/var/run/docker.sock"
        LogConfiguration:
          LogDriver: awslogs
          Options:
            awslogs-group:
              Fn::Sub: ${AWS::StackName}/ecs/WorkerService/slave
            awslogs-region: { "Ref": "AWS::Region" }
            awslogs-stream-prefix: docker
        Environment:
          - Name: "JAVA_TOOL_OPTIONS"
            Value:
              Fn::Sub: "-Xmx${JenkinsSlaveMaxHeapSize} -Xms${JenkinsSlaveInitialHeapSize}"
          - Name: "DOCKER_GID"
            Value: { "Ref": "DockerGroupId" }
          - Name: "JENKINS_USERNAME"
            Value: { "Ref": "JenkinsUsername" }
          - Name: "KMS_JENKINS_PASSWORD"
            Value: { "Ref": "JenkinsPassword" }
          - Name: "JENKINS_SLAVE_LABELS"
            Value: "docker"
          - Name: "JENKINS_URL"
            Value:
              Fn::Sub: "https://jenkins.${ApplicationDomain}/"
  JenkinsService:
    Type: "AWS::ECS::Service"
    DependsOn:
      - "ApplicationAutoscaling"
    Properties:
      Cluster: { "Ref": "ApplicationCluster" }
      TaskDefinition: { "Ref": "JenkinsTaskDefinition" }
      DesiredCount: 1
      DeploymentConfiguration:
        MinimumHealthyPercent: 0
        MaximumPercent: 100
      LoadBalancers:
        - ContainerName: "jenkins"
          ContainerPort: { "Ref": "JenkinsMasterPort"}
          LoadBalancerName: {"Ref": "ApplicationLoadBalancer"}
      Role: { "Ref": "EcsServiceRole" }
  WorkerService:
    Type: "AWS::ECS::Service"
    DependsOn:
      - "WorkerAutoscaling"
      - "JenkinsService"
    Properties:
      Cluster: { "Ref": "WorkerCluster" }
      TaskDefinition: { "Ref": "WorkerTaskDefinition" }
      DesiredCount: { "Ref": "WorkerServiceDesiredCount"}
      DeploymentConfiguration:
        MinimumHealthyPercent:
          Fn::If:
            - "WorkerSingleInstanceCondition"
            - 0
            - 50
        MaximumPercent: 200
  EcsServiceRole:
    Type: "AWS::IAM::Role"
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: "Allow"
            Principal:
              Service:
                - "ecs.amazonaws.com"
            Action:
              - "sts:AssumeRole"
      ManagedPolicyArns:
        - "arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceRole"
  ApplicationDmesgLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ec2/ApplicationAutoscaling/var/log/dmesg
      RetentionInDays: { "Ref": LogRetention }
  ApplicationDockerLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ec2/ApplicationAutoscaling/var/log/docker
      RetentionInDays: { "Ref": LogRetention }
  ApplicationEcsAgentLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ec2/ApplicationAutoscaling/var/log/ecs/ecs-agent
      RetentionInDays: { "Ref": LogRetention }
  ApplicationEcsInitLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ec2/ApplicationAutoscaling/var/log/ecs/ecs-init
      RetentionInDays: { "Ref": LogRetention }
  ApplicationMessagesLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ec2/ApplicationAutoscaling/var/log/messages
      RetentionInDays: { "Ref": LogRetention }
  WorkerDmesgLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ec2/WorkerAutoscaling/var/log/dmesg
      RetentionInDays: { "Ref": LogRetention }
  WorkerDockerLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ec2/WorkerAutoscaling/var/log/docker
      RetentionInDays: { "Ref": LogRetention }
  WorkerEcsAgentLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ec2/WorkerAutoscaling/var/log/ecs/ecs-agent
      RetentionInDays: { "Ref": LogRetention }
  WorkerEcsInitLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ec2/WorkerAutoscaling/var/log/ecs/ecs-init
      RetentionInDays: { "Ref": LogRetention }
  WorkerMessagesLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ec2/WorkerAutoscaling/var/log/messages
      RetentionInDays: { "Ref": LogRetention }
  JenkinsServiceLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ecs/JenkinsService/jenkins
      RetentionInDays: { "Ref": LogRetention }
  WorkerServiceLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ecs/WorkerService/slave
      RetentionInDays: { "Ref": LogRetention }
```

4\. Once the playbook execution completes successfully, login to the **demo-resources** account in the AWS console.  You should see a new CloudFormation stack called **jenkins-demo** in the **CloudFormation** dashboard:

![Jenkins Demo Stack](/images/jenkins-stack.png)

5\. If you navigate to **ECS > Clusters** you should be able to see the ECS cluster for the Jenkins application.

![Jenkins ECS Cluster](/images/jenkins-clusters.png)

6\. You can verify the jenkins is functional by browsing to `https://jenkins.demo.cloudhotspot.co/`:

![Browsing to Jenkins application](/images/jenkins-deployed.png)

## Wrap Up

We first published a Docker image required for the Jenkins stack to the ECR repository created earlier in this tutotial. We then created a new environment in the jenkns aws playbook, learned how to encrypt secrets using the AWS KMS key created earlier in the CloudFormation resources stack, and finally successully deployed the Jenkins stack.
