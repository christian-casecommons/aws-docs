---
date: 2017-03-21T01:37:17+13:00
title: Frontend Stack
weight: 81
---

## Introduction

While have a created web-facing application when orchestrating the [intake accelerator stack](/aws-docs/intake-accelerator/), lets create simpler, "static" application, fronted by cloudfront.

## Creating the Playbook

We will get started by cloning the [frontend-aws](https://github.com/casecommons/frontend-aws) repository that defines attributes needed to deploy a static, web-facing application.

1\. Clone the [frontend-aws](https://github.com/casecommons/frontend-aws) repostiory.

```bash
$ git clone git@github.com:casecommons/frontend-aws.git
  Cloning into 'frontend-aws'...
  remote: Counting objects: 31, done.
  remote: Compressing objects: 100% (18/18), done.
  remote: Total 31 (delta 6), reused 29 (delta 4), pack-reused 0
  Receiving objects: 100% (31/31), 5.70 KiB | 0 bytes/s, done.
  Resolving deltas: 100% (6/6), done.
$ cd frontend-aws
$ git checkout -b feature/new-site
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

4\.  Modify the `group_vars/cbp-dev/vars.yml` file; if you are defining a new DNS entry, please follow the directions for creating a [ca certificate](/aws-docs/intake-api/#creating-a-certificate).

{{< highlight python "hl_lines=3" >}}
Sts.Role: arn:aws:iam::334274607422:role/admin

# aws-cloudformation settings
Config.Frontend:
  SubDomain: frontend
  DomainName: dev.casebookplatform.org
  AcmCertificateArn: arn:aws:acm:us-east-1:334274607422:certificate/5e36d6bf-bf6d-4879-a3c9-339c7f10e156
  AlarmErrorRateEnabledCondition: "false"
  AlarmErrorRateThreshold: "5"
  AlarmErrorRatePeriod: "60"
  Region: "{{ Sts.Region }}"
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

{{% note title="Note" %}}
Orchestrating a cloudfront distribution takes ~20 minutes
{{% /note %}}

3\. Finally, make sure that the stack "frontend-aws" has successfully been created.

![Stack Image](/images/frontend-aws-stack.png)

## Creating a DNS record

We have setup a cloudfront distrubution and an S3 store, but we need to create a DNS record to point to the newly minted distribution.

1\. Clone the [dns-configuration](https://github.com/casecommons/dns-configuration) repostiory.

```bash
$ git clone git@github.com:casecommons/dns-configuration.git
  Cloning into 'dns-configuration'...
  remote: Counting objects: 27, done.
  remote: Compressing objects: 100% (18/18), done.
  remote: Total 27 (delta 13), reused 21 (delta 7), pack-reused 0
  Receiving objects: 100% (27/27), 5.31 KiB | 0 bytes/s, done.
  Resolving deltas: 100% (13/13), done.
$ cd dns-configuration
$ git checkout -b feature/new-feature
```

2\. Modify the file `./group_vars/cbp-dev/vars.yml` add add the cloudfront distro domain

```yaml
---
Sts.Role: arn:aws:iam::334274607422:role/admin

Config.Zones:
  DevCasebookplatformOrg:
    Name: dev.casebookplatform.org
    Create: False
    Records:
      - Name: frontend
        Type: CNAME
        TTL: 600
        Values:
          - "dbo04bytssmgx.cloudfront.net"
```

3\. Run the Ansible playbook targeting the `cbp-dev` environment as demonstrated below:

```bash
$ ansible-playbook site.yml -e env=$AWS_PROFILE
```

4\. Ensure that CNAME record has been created:

![Stack Image](/images/frontend-aws-route53.png)


## Deploying the application

We have setup the infrastructure to host a static application; next, we must deploy the application itself to S3.

1\. Clone the [react-template-app](https://github.com/casecommons/react-template-app) repostiory.

```bash
$ git clone git@github.com:casecommons/react-template-app.git
  Cloning into 'react-template-app'...
  remote: Counting objects: 27, done.
  remote: Compressing objects: 100% (18/18), done.
  remote: Total 27 (delta 13), reused 21 (delta 7), pack-reused 0
  Receiving objects: 100% (27/27), 5.31 KiB | 0 bytes/s, done.
  Resolving deltas: 100% (13/13), done.
$ cd react-template-app
$ git checkout -b feature/new-site
```

2\.  Modify the `./Makefile` file.

{{< highlight python "hl_lines=3" >}}
BUCKET ?= frontend.dev.casebookplatform.org
REPOSITORY ?= git@github.com:coryhouse/react-slingshot
{{< /highlight >}}

3\. Build the release artifact that will be deployed.

```bash
$ make release
  => Creating build artifacts..
  ...
```

4\. Ensure your local AWS environment is configured to target the **cbp-dev** account:

```bash
$ export AWS_PROFILE=cbp-dev
```

5\. Publish the build artifact to S3

```bash
$ make publish
  => Uploading static site to s3
  aws s3 cp \
      --region us-east-1 \
      --recursive \
      --include=* \
        ./build/react-slingshot/dist/ s3://frontend.dev.casebookplatform.org/
  upload: build/react-slingshot/dist/favicon.ico to s3://frontend.dev.casebookplatform.org/favicon.ico
  upload: build/react-slingshot/dist/main.fd96d68761e3952f22c42f7f8b76fffc.css to s3://frontend.dev.casebookplatform.org/main.fd96d68761e3952f22c42f7f8b76fffc.css
  upload: build/react-slingshot/dist/main.fd96d68761e3952f22c42f7f8b76fffc.css.map to s3://frontend.dev.casebookplatform.org/main.fd96d68761e3952f22c42f7f8b76fffc.css.map
  upload: build/react-slingshot/dist/index.html to s3://frontend.dev.casebookplatform.org/index.html
  upload: build/react-slingshot/dist/main.55fefde33b4974a35de6.js.map to s3://frontend.dev.casebookplatform.org/main.55fefde33b4974a35de6.js.map
  upload: build/react-slingshot/dist/main.55fefde33b4974a35de6.js to s3://frontend.dev.casebookplatform.org/main.55fefde33b4974a35de6.js
```

6\. Finally, direct a browser to the [frontend application](http://frontend.dev.casebookplatform.org)

![Browser Image](/images/frontend-aws-browser.png)

## Wrap Up

We have added the necessary infrastructure to host a static application and have published the application to an S3 bucket, which is then served by a cloudfront distribution.
