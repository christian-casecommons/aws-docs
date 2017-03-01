---
date: 2017-02-20T06:39:45+13:00
title: CloudFormation Resources
weight: 30
---

## Introduction

The [aws-cloudformation](https://github.com/casecommons/aws-cloudformation) role includes a [CloudFormation resources template](https://github.com/casecommons/aws-cloudformation/blob/master/templates/cfn.yml.j2) that is designed to define the following AWS resources:

- S3 bucket for CloudFormation templates
- S3 bucket for Lambda functions used to support CloudFormation custom resources
- Key Management Service (KMS) Key used for secrets management in CloudFormation stacks

By default, without any custom configuration, the CloudFormation resources template will create each of the above resources.

The template currently includes some optional configuration parameters as described below:

- `config_cfn_lambda_remote_accounts` - optional list of accounts that the S3 bucket for Lambda functions will be shared with.
- `config_cfn_kms_admins` - optional list of IAM roles that have administration level access to manage the KMS key.  
By default the **admin** role created by the [security resources playbook]({{< relref "security-resources/index.md" >}}) is granted this access.
- `config_cfn_kms_encrypters` - optional list of IAM roles that have permission to encrypt ciphertext using the KMS key.  
By default the **admin** role created by the [security resources playbook]({{< relref "security-resources/index.md" >}}) is granted this access.
- `config_cfn_kms_decrypters` - optional list of IAM roles that have permission to decrypt ciphertext using the KMS key.  
By default the **admin** role created by the [security resources playbook]({{< relref "security-resources/index.md" >}}) is granted this access.

{{< note title="Note" >}}
Before commencing the tasks below, ensure you have successfully completed all tasks in [Security Resources]({{< relref "security-resources/index.md" >}}).
{{< /note >}}

## Creating the Playbook

We will get started by establishing a CloudFormation resources playbook that defines CloudFormation resources for **Demo Resources** account.

1\. Clone the [AWS Starter](https://github.com/casecommons/aws-starter) to a local folder called `demo-cloudformation-resources` and the re-initialise the Git repository.

```bash
$ git clone git@github.com:casecommons/aws-starter.git demo-cloudformation-resources
  Cloning into 'demo-cloudformation-resources'...
  remote: Counting objects: 22, done.
  remote: Compressing objects: 100% (14/14), done.
  remote: Total 22 (delta 4), reused 22 (delta 4), pack-reused 0
  Receiving objects: 100% (22/22), done.
  Resolving deltas: 100% (4/4), done
$ cd demo-cloudformation-resources
$ rm -rf .git
$ git init
Initialized empty Git repository in /Users/jmenga/Source/casecommons/demo-cloudformation-resources/.git/
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
- extracting aws-cloudformation to /Users/jmenga/Source/casecommons/demo-cloudformation-resources/roles/aws-cloudformation
- aws-cloudformation was installed successfully
- extracting aws-sts to /Users/jmenga/Source/casecommons/demo-cloudformation-resources/roles/aws-sts
- aws-sts was installed successfully
{{< /highlight >}}

4\.  Modify the `group_vars/all/vars.yml` file, which contains global settings for the playbook:

{{< highlight python "hl_lines=3" >}}
# Stack Settings
cf_stack_name: cloudformation-resources
cf_stack_template: "templates/cfn.yml.j2"
cf_stack_tags:
  org:business:owner: Casecommons
  org:business:product: CloudFormation Resources
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

Notice that we reference the [templates/cfn.yml.y2 template](https://github.com/casecommons/aws-cloudformation/blob/master/templates/cfn.yml.j2) that is embedded within the [aws-cloudformation role](https://github.com/casecommons/aws-cloudformation).

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

4\. Finally add the `sts_role_arn` setting to `group_vars/demo-resources/vars.yml`:

```python
# STS role in the target AWS account to assume
sts_role_arn: "arn:aws:iam::094411466117:role/admin"
```

## Running the Playbook

The CloudFormation template in the `aws-cloudformation` role will create the following resources by default:

- An S3 bucket for CloudFormation templates
- An S3 bucket for Lambda functions used to support CloudFormation custom resources
- A Key Management Service (KMS) Key used for secrets management in CloudFormation stacks

Let's now run the playbook to create these resources.  

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
└── 20170226195006
    ├── cloudformation-resources-config.json
    ├── cloudformation-resources-policy.json
    ├── cloudformation-resources-stack.json
    └── cloudformation-resources-stack.yml
```

The following shows the `cloudformation-resources-stack.yml` file that was generated and uploaded to CloudFormation:

```yaml
AWSTemplateFormatVersion: "2010-09-09"

Description: CloudFormation Resources

Resources:
  CfnTemplatesBucket:
    Type: "AWS::S3::Bucket"
    Properties:
      BucketName:
        Fn::Sub: "${AWS::AccountId}-cfn-templates"
      Tags:
        - Key: Name
          Value: 
            Fn::Sub: "${AWS::AccountId}-cfn-templates"
  CfnLambdaBucket:
    Type: "AWS::S3::Bucket"
    Properties:
      BucketName: 
        Fn::Sub: "${AWS::AccountId}-cfn-lambda"
      VersioningConfiguration:
        Status: "Enabled"
      Tags:
        - Key: Name
          Value:
            Fn::Sub: "${AWS::AccountId}-cfn-lambda"
  CfnMasterKey:
    Type: "AWS::KMS::Key"
    Properties:
      Description: 
        Fn::Sub: "Credential Store Master Key for ${AWS::StackName}"
      Enabled: "true"
      KeyPolicy:
        Version: "2012-10-17"
        Id: "key-policy"
        Statement: 
          - Sid: "Allow root account access to key"
            Effect: "Allow"
            Principal:
              AWS:
                Fn::Sub: "arn:aws:iam::${AWS::AccountId}:root"
            Action: "kms:*"
            Resource: "*"
          - Sid: "Allow administration of the key"
            Effect: "Allow"
            Principal:
              AWS:
              - Fn::Sub: "arn:aws:iam::${AWS::AccountId}:role/admin"
            Action:
            - "kms:Create*"
            - "kms:Describe*"
            - "kms:Enable*"
            - "kms:List*"
            - "kms:Put*"
            - "kms:Update*"
            - "kms:Revoke*"
            - "kms:Disable*"
            - "kms:Get*"
            - "kms:Delete*"
            - "kms:ScheduleKeyDeletion"
            - "kms:CancelKeyDeletion"
            Resource: "*"
          - Sid: "Allow encryption use"
            Effect: "Allow"
            Principal:
              AWS:
              - Fn::Sub: "arn:aws:iam::${AWS::AccountId}:role/admin"
            Action:
            - "kms:Encrypt"
            - "kms:ReEncrypt*"
            - "kms:GenerateDataKey*"
            - "kms:DescribeKey"
            Resource: "*"
          - Sid: "Allow decryption use"
            Effect: "Allow"
            Principal:
              AWS:
              - Fn::Sub: "arn:aws:iam::${AWS::AccountId}:role/admin"
            Action:
            - "kms:Decrypt"
            - "kms:DescribeKey"
            Resource: "*"
Outputs:
  CfnTemplatesBucketName:
    Description: "CloudFormation Templates Bucket Name"
    Value:
      Fn::Sub: "${CfnTemplatesBucket.DomainName}"
    Export:
      Name: "CfnTemplatesBucket"
  CfnTemplatesBucketURL:
    Description: "CloudFormation Templates Bucket URL"
    Value: 
      Fn::Sub: "${CfnTemplatesBucket.WebsiteURL}"
    Export:
      Name: "CfnTemplatesBucketURL"
  CfnMasterKey:
    Description: "CloudFormation Master Key ID"
    Value: 
      Fn::Sub: "${CfnMasterKey}"
    Export:
      Name: "CfnMasterKey"
  CfnMasterKeyArn:
    Description: "CloudFormation Master Key ARN"
    Value: 
      Fn::Sub: "${CfnMasterKey.Arn}"
    Export:
      Name: "CfnMasterKeyArn"
  CfnLambdaBucketName:
    Description: "CloudFormation Lambda Bucket Name"
    Value: 
      Fn::Sub: "${CfnLambdaBucket.DomainName}"
    Export:
      Name: "CfnLambdaBucket"
  CfnLambdaBucketURL:
    Description: "CloudFormation Lambda Bucket URL"
    Value: 
      Fn::Sub: "${CfnLambdaBucket.WebsiteURL}"
    Export:
      Name: "CfnLambdaBucketURL"
```

4\. Once the playbook execution completes successfully, login to the **demo-resources** account.  You should see a new CloudFormation stack called `cloudformation-resources`:

![CloudFormation Resources Stack](/images/cloudformation-resources.png)

Notice that two S3 buckets and a KMS key have been created.  

5\. Select the **CloudFormation** dropdown and click on **Exports**:

![Selecting CloudFormation Exports](/images/cloudformation-exports.png)

6\. Notice that a number of CloudFormation exports have been created by both the **security-resources** and **cloudformation-resources** stacks.  These exports can be referenced in other stacks by their logical **Export Name** using the CloudFormation [Fn::ImportValue intrinsic function](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-importvalue.html), making it easier to consume these resources.

![CloudFormation Exports](/images/cfn-cloudformation-exports.png)

## Wrap Up

We created CloudFormation resources in our **demo-resources** account, which provide the supporting resources for creating additional CloudFormation stacks that require features such as CloudFormation custom resources and secrets management.

{{< note title="Note" >}}
At this point you should commit your changes to the CloudFormation resources playbook before continuing.
{{< /note >}}
