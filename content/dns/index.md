---
date: 2017-03-21T01:37:17+13:00
title: DNS Configuration
weight: 82
---

## Introduction

We have implicitely configured hosted-zone records as an aspect of building other stacks, but in the case of explicitely codifying our hosted zone and its associated records, we can turn to the dns-configuration repository.


## Creating the Playbook

We will get started by cloning the dns-configuration repository that defines hosted zones and their associated records

1\. Clone the [dns-configuration](https://github.com/casecommons/dns-configuration) repostiory.

```bash
$ git clone git@github.com:casecommons/dns-configuration.git
  Cloning into 'dns-configuration'...
  remote: Counting objects: 19, done.
  remote: Compressing objects: 100% (13/13), done.
  remote: Total 19 (delta 1), reused 18 (delta 0), pack-reused 0
  Receiving objects: 100% (19/19), done.
  Resolving deltas: 100% (1/1), done.
$ cd dns-configuration
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

Config.Zones:
  DevCasebookplatformOrg:
    Name: dev.casebookplatform.org
    Records:
      - Name: test
        Type: A
        TTL: 14400
        Values:
          - 192.252.144.15
      - Name: ""
        Type: MX
        TTL: 3600
        Values:
          - 10 aspmx.l.google.com
          - 20 alt1.aspmx.l.google.com
          - 30 alt2.aspmx.l.google.com
          - 40 aspmx2.googlemail.com
          - 50 aspmx3.googlemail.com
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

3\. Make sure that the stack "dns-configuration" has successfully been created.

![Stack Image](/images/dns-configuration-stack.png)


## Wrap Up

We have added two records to the hosted zone, dev.casebookorg.platform, and done so in a controled and codified fashion.
