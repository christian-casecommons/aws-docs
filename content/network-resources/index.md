---
date: 2017-02-20T06:39:45+13:00
title: Network Resources
weight: 40
---

## Introduction

The [aws-cloudformation](https://github.com/casecommons/aws-cloudformation) role includes a [network resources template](https://github.com/casecommons/aws-cloudformation/blob/master/templates/network.yml.j2) that is designed to define the following AWS resources:

- Subnets
- Route Tables
- Internet Gateways
- DHCP Option Sets
- VPC Endpoints
- VPC Flow Logs
- Route53 Public Zones
- Route53 Private Zones

{{% note title="Note" %}}
Before commencing the tasks below, ensure you have successfully completed all tasks in the following sections:

- [Security Resources]({{< relref "security-resources/index.md" >}})
- [CloudFormation Resources]({{< relref "cloudformation-resources/index.md" >}})
{{% /note %}}

## Creating the Playbook

We will get started by establishing a network resources playbook that defines network resources for **Demo Resources** account.

1\. Clone the [AWS Starter](https://github.com/casecommons/aws-starter) to a local folder called `demo-network-resources` and the re-initialise the Git repository.

```bash
$ git clone git@github.com:casecommons/aws-starter.git demo-network-resources
  Cloning into 'demo-network-resources'...
  remote: Counting objects: 22, done.
  remote: Compressing objects: 100% (14/14), done.
  remote: Total 22 (delta 4), reused 22 (delta 4), pack-reused 0
  Receiving objects: 100% (22/22), done.
  Resolving deltas: 100% (4/4), done
$ cd demo-network-resources
$ rm -rf .git
$ git init
Initialized empty Git repository in /Users/jmenga/Source/casecommons/demo-network-resources/.git/
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
- extracting aws-cloudformation to /Users/jmenga/Source/casecommons/demo-network-resources/roles/aws-cloudformation
- aws-cloudformation was installed successfully
- extracting aws-sts to /Users/jmenga/Source/casecommons/demo-network-resources/roles/aws-sts
- aws-sts was installed successfully
{{< /highlight >}}

4\.  Modify the `group_vars/all/vars.yml` file, which contains global settings for the playbook:

{{< highlight python "hl_lines=3" >}}
# Stack Settings
cf_stack_name: network-resources
cf_stack_template: "templates/network.yml.j2"
cf_stack_tags:
  org:business:owner: Casecommons
  org:business:product: Network Resources
  org:business:severity: High
  org:tech:environment: "{{ env }}"
  org:tech:contact: pema@casecommons.org
  
# CloudFormation settings
# This sets a policy that disables updates to existing resources
# This requires you to explicitly set the cf_disable_stack_policy flag to true when running the playbook
cf_stack_policy:
  Statement:
  - Effect: "Deny"
    Action: "Update:*"
    Principal: "*"
    Resource: "*"
{{< /highlight >}}

Notice that we reference the [templates/network.yml.y2 template](https://github.com/casecommons/aws-cloudformation/blob/master/templates/network.yml.j2) that is embedded within the [aws-cloudformation role](https://github.com/casecommons/aws-cloudformation).

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

# VPC CIDR Block
config_vpc_cidr: 192.168.16.0/20

# VPC Domain Settings
# This will configure a private Route53 zone of <vpc-id>.<config_vpc_root_domain>
config_vpc_root_domain: casecommons.org

# AZ Count
config_az_count: 3

# Public Route53 Zones
config_public_domains:
  - demo.cloudhotspot.co
```

Here we target the **demo-resources** account by specifying the **demo-resources** IAM **admin** role in the `sts_role_arn` variable, whilst the following settings customize the network resources that will be created:

- `config_vpc_cidr` - defines the CIDR block for the VPC that will be created.  The CIDR block must be large enough to create the required number of subnets, which can be calculated by multiplying the number of security zones (4 by default) by the number of availability zones (3 as defined by `config_az_count`) - i.e. 12 subnets with a default size/subnet mask of /24 or 255.255.255.0.  The value of `192.168.16.0/20` will accommodate 16 x /24 subnets, so is sufficiently sized.

- `config_vpc_root_domain` - when configured creates a Route53 private zone in the format `<vpc-id>.<config_vpc_root_domain>`.  In this example, if the VPC ID is `vpc-1234567` then the configuration above would create a private zone of `vpc-1234567.casecommons.org`.

- `config_az_count` - defines the number of availability zones to create subnets in.

- `config_public_domains` - defines a list of public Route53 domains or zones to create 

## Running the Playbook

Now that we've defined environment settings for the **demo-resources** environment, let's run the playbook to create network resources for the environment.  

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

```bash
$ tree build
build
└── 20170226211011
    ├── network-resources-config.json
    ├── network-resources-policy.json
    ├── network-resources-stack.json
    └── network-resources-stack.yml
```

The following shows the `network-resources-stack.yml` file that was generated and uploaded to CloudFormation:

```python
AWSTemplateFormatVersion: "2010-09-09"

Description: Core Network Resources
    
Resources:
  Vpc:
    Type: "AWS::EC2::VPC"
    Properties:
      CidrBlock: "192.168.16.0/20"
      EnableDnsSupport: True
      EnableDnsHostnames: True
      Tags:
        - Key: "Name"
          Value: "default-vpc"
  InternetGateway:
    Type: "AWS::EC2::InternetGateway"
    Properties: 
      Tags:
        - Key: "Name"
          Value: "default-igw"
        - Key: "org:security:level"
          Value: "public"
  InternetGatewayAttachment:
    Type: "AWS::EC2::VPCGatewayAttachment"
    Properties:
      InternetGatewayId: { "Ref": "InternetGateway" }
      VpcId:
        Ref: Vpc
  PublicRouteTable:
    Type: "AWS::EC2::RouteTable"
    Properties:
      VpcId:
        Ref: Vpc
      Tags:
        - Key: "Name"
          Value: "default-public-route-table"
        - Key: "org:security:level"
          Value: "public"
  PrivateRouteTable:
    Type: "AWS::EC2::RouteTable"
    Properties:
      VpcId:
        Ref: Vpc
      Tags:
        - Key: "Name"
          Value: "default-private-route-table"
        - Key: "org:security:level"
          Value: "mixed"
  PublicDefaultRoute:
    Type: "AWS::EC2::Route"
    DependsOn: "InternetGatewayAttachment"
    Properties:
      DestinationCidrBlock: "0.0.0.0/0"
      GatewayId: { "Ref": "InternetGateway" }
      RouteTableId: { "Ref": "PublicRouteTable" }
  S3Endpoint:
    Type: "AWS::EC2::VPCEndpoint"
    Properties:
      RouteTableIds: 
        - { "Ref" : "PublicRouteTable" }
        - { "Ref" : "PrivateRouteTable" }
      ServiceName: 
        Fn::Sub: "com.amazonaws.${AWS::Region}.s3"
      VpcId:
        Ref: Vpc
  VpcFlowLog:
    Type: "AWS::EC2::FlowLog"
    Properties:
      DeliverLogsPermissionArn:
        Fn::Sub: "arn:aws:iam::${AWS::AccountId}:role/${VpcFlowLogRole}"
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/vpc/FlowLog
      ResourceId: { "Ref": "Vpc" }
      ResourceType: "VPC"
      TrafficType: "ALL"
  VpcFlowLogRole:
    Type: "AWS::IAM::Role"
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: "Allow"
            Principal: 
              Service: [ "vpc-flow-logs.amazonaws.com" ]
            Action: [ "sts:AssumeRole" ]
      Path: "/"
      Policies:
      - PolicyName: CloudWatchLogs
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
            - Effect: Allow
              Action:
              - "logs:CreateLogGroup"
              - "logs:CreateLogStream"
              - "logs:PutLogEvents"
              - "logs:DescribeLogGroups"
              - "logs:DescribeLogStreams"
              Resource:
                Fn::Sub: "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:${AWS::StackName}/vpc/FlowLog*"
  VpcFlowLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName: 
        Fn::Sub: ${AWS::StackName}/vpc/FlowLog
      RetentionInDays: 7
  VpcPrivateZone:
    Type: "AWS::Route53::HostedZone"
    Properties: 
      HostedZoneConfig: 
        Comment: 
          Fn::Sub: "${Vpc} private zone"
      HostedZoneTags:
        - Key: "Name"
          Value: 
            Fn::Sub: "${Vpc}.casecommons.org"
        - Key: "org:security:level"
          Value: "mixed"
      Name:
        Fn::Sub: "${Vpc}.casecommons.org"
      VPCs:
        - VPCId: { "Ref": "Vpc" }
          VPCRegion: { "Ref": "AWS::Region" }
  DemoCloudhotspotCoZone:
      Type: "AWS::Route53::HostedZone"
      Properties: 
        HostedZoneConfig: 
          Comment: "demo.cloudhotspot.co zone"
        HostedZoneTags:
          - Key: "Name"
            Value: "demo.cloudhotspot.co"
          - Key: "org:security:level"
            Value: "mixed"
        Name: "demo.cloudhotspot.co"
  PublicSubnetA:
    Type: "AWS::EC2::Subnet"
    Properties:
      VpcId:
        Ref: Vpc
      CidrBlock: "192.168.16.0/24"
      AvailabilityZone: 
        Fn::Sub: "${AWS::Region}a"
      Tags:
        - Key: "Name"
          Value: "public-a"
        - Key: "org:security:level"
          Value: "public"
  PublicSubnetARouting:
    Type: "AWS::EC2::SubnetRouteTableAssociation"
    Properties:
      RouteTableId: { "Ref": "PublicRouteTable" }
      SubnetId: { "Ref": "PublicSubnetA" }
  PublicSubnetB:
    Type: "AWS::EC2::Subnet"
    Properties:
      VpcId:
        Ref: Vpc
      CidrBlock: "192.168.17.0/24"
      AvailabilityZone: 
        Fn::Sub: "${AWS::Region}b"
      Tags:
        - Key: "Name"
          Value: "public-b"
        - Key: "org:security:level"
          Value: "public"
  PublicSubnetBRouting:
    Type: "AWS::EC2::SubnetRouteTableAssociation"
    Properties:
      RouteTableId: { "Ref": "PublicRouteTable" }
      SubnetId: { "Ref": "PublicSubnetB" }
  PublicSubnetC:
    Type: "AWS::EC2::Subnet"
    Properties:
      VpcId:
        Ref: Vpc
      CidrBlock: "192.168.18.0/24"
      AvailabilityZone: 
        Fn::Sub: "${AWS::Region}c"
      Tags:
        - Key: "Name"
          Value: "public-c"
        - Key: "org:security:level"
          Value: "public"
  PublicSubnetCRouting:
    Type: "AWS::EC2::SubnetRouteTableAssociation"
    Properties:
      RouteTableId: { "Ref": "PublicRouteTable" }
      SubnetId: { "Ref": "PublicSubnetC" }
  MediumSubnetA:
    Type: "AWS::EC2::Subnet"
    Properties:
      VpcId:
        Ref: Vpc
      CidrBlock: "192.168.19.0/24"
      AvailabilityZone: 
        Fn::Sub: "${AWS::Region}a"
      Tags:
        - Key: "Name"
          Value: "medium-a"
        - Key: "org:security:level"
          Value: "medium"
  MediumSubnetARouting:
    Type: "AWS::EC2::SubnetRouteTableAssociation"
    Properties:
      RouteTableId: { "Ref": "PrivateRouteTable" }
      SubnetId: { "Ref": "MediumSubnetA" }
  MediumSubnetB:
    Type: "AWS::EC2::Subnet"
    Properties:
      VpcId:
        Ref: Vpc
      CidrBlock: "192.168.20.0/24"
      AvailabilityZone: 
        Fn::Sub: "${AWS::Region}b"
      Tags:
        - Key: "Name"
          Value: "medium-b"
        - Key: "org:security:level"
          Value: "medium"
  MediumSubnetBRouting:
    Type: "AWS::EC2::SubnetRouteTableAssociation"
    Properties:
      RouteTableId: { "Ref": "PrivateRouteTable" }
      SubnetId: { "Ref": "MediumSubnetB" }
  MediumSubnetC:
    Type: "AWS::EC2::Subnet"
    Properties:
      VpcId:
        Ref: Vpc
      CidrBlock: "192.168.21.0/24"
      AvailabilityZone: 
        Fn::Sub: "${AWS::Region}c"
      Tags:
        - Key: "Name"
          Value: "medium-c"
        - Key: "org:security:level"
          Value: "medium"
  MediumSubnetCRouting:
    Type: "AWS::EC2::SubnetRouteTableAssociation"
    Properties:
      RouteTableId: { "Ref": "PrivateRouteTable" }
      SubnetId: { "Ref": "MediumSubnetC" }
  HighSubnetA:
    Type: "AWS::EC2::Subnet"
    Properties:
      VpcId:
        Ref: Vpc
      CidrBlock: "192.168.22.0/24"
      AvailabilityZone: 
        Fn::Sub: "${AWS::Region}a"
      Tags:
        - Key: "Name"
          Value: "high-a"
        - Key: "org:security:level"
          Value: "high"
  HighSubnetARouting:
    Type: "AWS::EC2::SubnetRouteTableAssociation"
    Properties:
      RouteTableId: { "Ref": "PrivateRouteTable" }
      SubnetId: { "Ref": "HighSubnetA" }
  HighSubnetB:
    Type: "AWS::EC2::Subnet"
    Properties:
      VpcId:
        Ref: Vpc
      CidrBlock: "192.168.23.0/24"
      AvailabilityZone: 
        Fn::Sub: "${AWS::Region}b"
      Tags:
        - Key: "Name"
          Value: "high-b"
        - Key: "org:security:level"
          Value: "high"
  HighSubnetBRouting:
    Type: "AWS::EC2::SubnetRouteTableAssociation"
    Properties:
      RouteTableId: { "Ref": "PrivateRouteTable" }
      SubnetId: { "Ref": "HighSubnetB" }
  HighSubnetC:
    Type: "AWS::EC2::Subnet"
    Properties:
      VpcId:
        Ref: Vpc
      CidrBlock: "192.168.24.0/24"
      AvailabilityZone: 
        Fn::Sub: "${AWS::Region}c"
      Tags:
        - Key: "Name"
          Value: "high-c"
        - Key: "org:security:level"
          Value: "high"
  HighSubnetCRouting:
    Type: "AWS::EC2::SubnetRouteTableAssociation"
    Properties:
      RouteTableId: { "Ref": "PrivateRouteTable" }
      SubnetId: { "Ref": "HighSubnetC" }
  ManagementSubnetA:
    Type: "AWS::EC2::Subnet"
    Properties:
      VpcId:
        Ref: Vpc
      CidrBlock: "192.168.25.0/24"
      AvailabilityZone: 
        Fn::Sub: "${AWS::Region}a"
      Tags:
        - Key: "Name"
          Value: "management-a"
        - Key: "org:security:level"
          Value: "management"
  ManagementSubnetARouting:
    Type: "AWS::EC2::SubnetRouteTableAssociation"
    Properties:
      RouteTableId: { "Ref": "PrivateRouteTable" }
      SubnetId: { "Ref": "ManagementSubnetA" }
  ManagementSubnetB:
    Type: "AWS::EC2::Subnet"
    Properties:
      VpcId:
        Ref: Vpc
      CidrBlock: "192.168.26.0/24"
      AvailabilityZone: 
        Fn::Sub: "${AWS::Region}b"
      Tags:
        - Key: "Name"
          Value: "management-b"
        - Key: "org:security:level"
          Value: "management"
  ManagementSubnetBRouting:
    Type: "AWS::EC2::SubnetRouteTableAssociation"
    Properties:
      RouteTableId: { "Ref": "PrivateRouteTable" }
      SubnetId: { "Ref": "ManagementSubnetB" }
  ManagementSubnetC:
    Type: "AWS::EC2::Subnet"
    Properties:
      VpcId:
        Ref: Vpc
      CidrBlock: "192.168.27.0/24"
      AvailabilityZone: 
        Fn::Sub: "${AWS::Region}c"
      Tags:
        - Key: "Name"
          Value: "management-c"
        - Key: "org:security:level"
          Value: "management"
  ManagementSubnetCRouting:
    Type: "AWS::EC2::SubnetRouteTableAssociation"
    Properties:
      RouteTableId: { "Ref": "PrivateRouteTable" }
      SubnetId: { "Ref": "ManagementSubnetC" }


Outputs:
  DefaultVpcId:
    Description: "Default VPC Identifier"
    Value:
      Ref: Vpc
    Export:
      Name: "DefaultVpcId"
  DefaultVpcCidr:
    Description: "Default VPC CIDR Block"
    Value: "192.168.16.0/20"
    Export:
      Name: "DefaultVpcCidr"
  DefaultVpcDnsServer:
    Description: "Default VPC AWS Provided DNS Server IP Address"
    Value: "192.168.16.2"
    Export:
      Name: "DefaultVpcDnsServer"
  DefaultVpcDomain:
    Description: "Default VPC Domain"
    Value: 
      Fn::Sub: "${Vpc}.casecommons.org"
    Export:
      Name: "DefaultVpcDomain"
  DefaultVpcZone:
    Description: "Default VPC Zone"
    Value: 
      Fn::Sub: "${Vpc}.casecommons.org."
    Export:
      Name: "DefaultVpcZone"
  DefaultPublicSubnetA:
    Description: "DefaultPublic Subnet Availability Zone A"
    Value: { "Ref": "PublicSubnetA" }
    Export:
      Name: "DefaultPublicSubnetA"
  DefaultPublicSubnetACidr:
    Description: "Default Public Subnet  Availability Zone A CIDR"
    Value: "192.168.16.0/24"
    Export:
      Name: "DefaultPublicSubnetACidr"
  DefaultPublicSubnetB:
    Description: "DefaultPublic Subnet Availability Zone B"
    Value: { "Ref": "PublicSubnetB" }
    Export:
      Name: "DefaultPublicSubnetB"
  DefaultPublicSubnetBCidr:
    Description: "Default Public Subnet  Availability Zone B CIDR"
    Value: "192.168.17.0/24"
    Export:
      Name: "DefaultPublicSubnetBCidr"
  DefaultPublicSubnetC:
    Description: "DefaultPublic Subnet Availability Zone C"
    Value: { "Ref": "PublicSubnetC" }
    Export:
      Name: "DefaultPublicSubnetC"
  DefaultPublicSubnetCCidr:
    Description: "Default Public Subnet  Availability Zone C CIDR"
    Value: "192.168.18.0/24"
    Export:
      Name: "DefaultPublicSubnetCCidr"
  DefaultMediumSubnetA:
    Description: "DefaultMedium Subnet Availability Zone A"
    Value: { "Ref": "MediumSubnetA" }
    Export:
      Name: "DefaultMediumSubnetA"
  DefaultMediumSubnetACidr:
    Description: "Default Medium Subnet  Availability Zone A CIDR"
    Value: "192.168.19.0/24"
    Export:
      Name: "DefaultMediumSubnetACidr"
  DefaultMediumSubnetB:
    Description: "DefaultMedium Subnet Availability Zone B"
    Value: { "Ref": "MediumSubnetB" }
    Export:
      Name: "DefaultMediumSubnetB"
  DefaultMediumSubnetBCidr:
    Description: "Default Medium Subnet  Availability Zone B CIDR"
    Value: "192.168.20.0/24"
    Export:
      Name: "DefaultMediumSubnetBCidr"
  DefaultMediumSubnetC:
    Description: "DefaultMedium Subnet Availability Zone C"
    Value: { "Ref": "MediumSubnetC" }
    Export:
      Name: "DefaultMediumSubnetC"
  DefaultMediumSubnetCCidr:
    Description: "Default Medium Subnet  Availability Zone C CIDR"
    Value: "192.168.21.0/24"
    Export:
      Name: "DefaultMediumSubnetCCidr"
  DefaultHighSubnetA:
    Description: "DefaultHigh Subnet Availability Zone A"
    Value: { "Ref": "HighSubnetA" }
    Export:
      Name: "DefaultHighSubnetA"
  DefaultHighSubnetACidr:
    Description: "Default High Subnet  Availability Zone A CIDR"
    Value: "192.168.22.0/24"
    Export:
      Name: "DefaultHighSubnetACidr"
  DefaultHighSubnetB:
    Description: "DefaultHigh Subnet Availability Zone B"
    Value: { "Ref": "HighSubnetB" }
    Export:
      Name: "DefaultHighSubnetB"
  DefaultHighSubnetBCidr:
    Description: "Default High Subnet  Availability Zone B CIDR"
    Value: "192.168.23.0/24"
    Export:
      Name: "DefaultHighSubnetBCidr"
  DefaultHighSubnetC:
    Description: "DefaultHigh Subnet Availability Zone C"
    Value: { "Ref": "HighSubnetC" }
    Export:
      Name: "DefaultHighSubnetC"
  DefaultHighSubnetCCidr:
    Description: "Default High Subnet  Availability Zone C CIDR"
    Value: "192.168.24.0/24"
    Export:
      Name: "DefaultHighSubnetCCidr"
  DefaultManagementSubnetA:
    Description: "DefaultManagement Subnet Availability Zone A"
    Value: { "Ref": "ManagementSubnetA" }
    Export:
      Name: "DefaultManagementSubnetA"
  DefaultManagementSubnetACidr:
    Description: "Default Management Subnet  Availability Zone A CIDR"
    Value: "192.168.25.0/24"
    Export:
      Name: "DefaultManagementSubnetACidr"
  DefaultManagementSubnetB:
    Description: "DefaultManagement Subnet Availability Zone B"
    Value: { "Ref": "ManagementSubnetB" }
    Export:
      Name: "DefaultManagementSubnetB"
  DefaultManagementSubnetBCidr:
    Description: "Default Management Subnet  Availability Zone B CIDR"
    Value: "192.168.26.0/24"
    Export:
      Name: "DefaultManagementSubnetBCidr"
  DefaultManagementSubnetC:
    Description: "DefaultManagement Subnet Availability Zone C"
    Value: { "Ref": "ManagementSubnetC" }
    Export:
      Name: "DefaultManagementSubnetC"
  DefaultManagementSubnetCCidr:
    Description: "Default Management Subnet  Availability Zone C CIDR"
    Value: "192.168.27.0/24"
    Export:
      Name: "DefaultManagementSubnetCCidr"
```

4\. Once the playbook execution completes successfully, login to the **demo-resources** account.  You should see a new CloudFormation stack called `network-resources`:

![Network Resources Stack](/images/network-resources.png)

Notice that a large number of network resources have been created.

5\. If you select the **CloudFormation** dropdown and click on **Exports**, notice that a large number of CloudFormation exports have been created by the **network-resources** stack.

![CloudFormation Exports](/images/network-resources-exports.png)

6\. If you open the VPC dashboard and select **Subnets**, a large number of subnets have now been created:

![Network Subnets](/images/network-subnets.png)

Notice that /24 subnets for availability zones A, B and C have been created in the **public**, **medium**, **high** and **management** security zones.

7\. If you open the Route53 dashboard and select **Hosted zones**, two Route53 zones have been created:

![Route53 Domains](/images/network-domains.png)

Notice that a public Route53 zone for `demo.cloudhotspot.co` has been created, along with a private Route53 zone for `<vpc-id>.casecommons.org`.

## Wrap Up

We created network resources in our **demo-resources** account, which provide the supporting network infrastructure for creating CloudFormation stacks for our applications.

{{< note title="Note" >}}
At this point you should commit your changes to the network resources playbook before continuing.
{{< /note >}}
