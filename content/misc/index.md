---
date: 2017-03-21T01:37:17+13:00
title: Miscellaneous
weight: 83
---

## Introduction

Although we have now finished a run-through of the platform, there are tidbits that defy easy categorization and which are presented below.

## Creating Certificates

Even though we [created](///aws-docs/intake-api/#creating-a-certificate) a certificate during the intake-api tutorial, as a part of the security stack, we are free to issue CA certificates as a part of any stack you should build in the future. As an example, we will start from the (now) familiar aws-starter kit.

1\. Clone the [aws-starter](https://github.com/casecommons/aws-starter) repostiory.

```bash
$ git clone git@github.com:casecommons/aws-starter.git
  Cloning into 'aws-starter'...
  remote: Counting objects: 19, done.
  remote: Compressing objects: 100% (13/13), done.
  remote: Total 19 (delta 1), reused 18 (delta 0), pack-reused 0
  Receiving objects: 100% (19/19), done.
  Resolving deltas: 100% (1/1), done.
$ cd aws-starter
$ git checkout -b update/configuration
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
- extracting aws-cloudformation to /Users/jmenga/Source/casecommons/demo-ecr-resources/roles/aws-cloudformation
- aws-cloudformation was installed successfully
- extracting aws-sts to /Users/jmenga/Source/casecommons/demo-ecr-resources/roles/aws-sts
- aws-sts was installed successfully
{{< /highlight >}}

4\.  Modify the `group_vars/cbp-dev/vars.yml` file

{{< highlight python "hl_lines=3" >}}
Sts.Role: arn:aws:iam::334274607422:role/admin
Stack.Template: templates/certificate.yml.j2
Stack.Name: test-certificates

Config.Certificates:
  DevCasebookPlatformOrg:
    DomainName: *.dev.casebookplatform.org
    SubjectAlternativeNames:
      - www.casebook.com
    ValidationOptions:
      - DomainName: *.casebookplatform.org
        ValidationDomain: casebookplatform.org
  ProdCasebookPlatformOrg:
    ...
{{< /highlight >}}


## Running the Playbook

Now that we've defined environment settings for the **demo-resources** environment, let's run the playbook to create ECR repository resources for the environment.

1\. Ensure your local AWS environment is configured to target the **cbp-dev** account:

```bash
$ export AWS_PROFILE=cbp-dev
```

2\. Run the Ansible playbook targeting the `cbp-dev` environment as demonstrated below:

{{< highlight bash >}}
$ ansible-playbook site.yml -e env=$AWS_PROFILE
{{< /highlight >}}

3\. Check the validation email address (administrator@casebookplatform.org) for a validation request for AWS and approve:

![Certificate Validation Email](/images/acm-email.png)

4\. Make sure that the stack has successfully been created.

![Stack Image](/images/aws-starter-stack.png)

## Wrap Up

We have added a wildcard CA certificate for the domain *.dev.casebookplatform.org; though we have presented doing so in a standalone repository, the same could be accomplished from a composite template to achieve the same effect.

