---
date: 2017-02-27T02:59:22+13:00
title: Intake API Stack
weight: 70
---

## Introduction

At this point, we have established all of the shared resource stacks in our **demo-users** and **demo-resources** accounts, and it is now time to focus on defining a playbook and CloudFormation stack for an application.

We will now deploy the Intake API application to our **demo-resources** account, creating a **demo** environment, with the Intake API application published at `https://demo-intake-api.demo.cloudhotspot.co`.

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

6\.  Note that because we are validating two domain names (the primary `*.demo.cloudhotspot.co` subject name and the `demo.cloudhotspot.co` subject alternative name), you must approve separate validation requests for each domain.

## Creating an Nginx Image

The Intake API stack uses a custom Nginx Docker image, which can be built by cloning the https://github.com/casecommons/docker-nginx.git repository locally.

Before we can create the Intake API stack, we need to publish this image to the **casecommons/nginx** ECR repository that was established when we created [ECR resources]({{< relref "ecr-resources/index.md" >}})

1\. Clone the Casecommons Docker Nginx repository to your local environment:

```bash
$ git clone https://github.com/casecommons/docker-nginx
Cloning into 'docker-nginx'...
remote: Counting objects: 48, done.
remote: Compressing objects: 100% (3/3), done.
remote: Total 48 (delta 0), reused 0 (delta 0), pack-reused 45
Receiving objects: 100% (48/48), 13.61 KiB | 0 bytes/s, done.
Resolving deltas: 100% (10/10), done.
$ cd docker-squid
$ tree -L 1
.
├── Makefile
├── Makefile.settings
├── README.md
├── docker
└── src
```

2\. Open the `Makefile` at the root of the **docker-nginx** repository and modify the highlighted settings:

{{< highlight make "hl_lines=3 4 5 6" >}}
# Project variables
export PROJECT_NAME ?= nginx
ORG_NAME ?= casecommons
REPO_NAME ?= nginx
DOCKER_REGISTRY ?= 160775127577.dkr.ecr.us-west-2.amazonaws.com
AWS_ACCOUNT_ID ?= 160775127577
DOCKER_LOGIN_EXPRESSION ?= eval $$(aws ecr get-login --registry-ids $(AWS_ACCOUNT_ID))
...
...
{{< /highlight >}}

Here we ensure the `Makefile` settings are configured with the correct Docker registry name (`DOCKER_REGISTRY`), organization name (`ORG_NAME`), repository name (`REPO_NAME`) and AWS account ID (`AWS_ACCOUNT_ID`).

3\. Run the `make login` command to login to the ECR repository for the **demo-resources** account:

```bash
$ export AWS_PROFILE=demo-resources-admin
$ make login
=> Logging in to Docker registry ...
Enter MFA code: *****
Login Succeeded
=> Logged in to Docker registry
```

4\. Run the `make release` command, which will build the squid image, and then `curl` the output endpoint to verify Nginx is working as expected:

```bash
$ make release
=> Building images...
Building nginx
Step 1/10 : FROM nginx:alpine
...
...
=> Build complete
=> Starting app service...
Creating network "nginx_default" with the default driver
Creating volume "nginx_webroot" with local driver
Creating nginx_app_1
=> Starting nginx service...
Creating nginx_nginx_1
=> Release environment created
=> Nginx is running at http://172.16.154.128:32796
$ curl -s http://172.16.154.128:32796
Hello World!
```

5\. Run the `make tag latest` command, which will tag the image with the `latest` tag:

```bash
$ make tag latest
=> Tagging release image with tags latest...
=> Tagging complete
```

5\. Run the `make publish` command, which will publish the image to your ECR repository:

```bash
$ make publish
=> Publishing release image to 160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/nginx...
The push refers to a repository [160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/nginx]
bacade763708: Pushed
2030c63749a0: Pushed
b0ef948bd579: Pushed
32418e8b3e29: Pushed
f6fb5b0f08d3: Pushed
cec0d431e110: Pushed
7cbcbac42c44: Pushed
latest: digest: sha256:98792f09c0199156ce31236705021e1d81de2ac969e06862d0dea6bce03a79d1 size: 1780
=> Publish complete
```

6\. Run the `make clean` command to clean up the local Docker environment.

```bash
$ make clean
=> Destroying release environment...
Stopping nginx_nginx_1 ... done
Stopping nginx_app_1 ... done
Removing nginx_nginx_1 ... done
Removing nginx_app_1 ... done
Removing network nginx_default
Removing volume nginx_webroot
=> Removing dangling images...
=> Clean complete
```

7\. In the AWS console under **ECS > Repositories**, you should now be able to see your newly published image in the **casecommons/nginx** repository:

![Nginx Image](/images/nginx-image.png)

## Creating an Elasticsearch Image

The Intake API stack requires Elasticsearch and creates an ECS cluster with that runs a single Elasticsearch container.

Although we will be using the official Elasticsearch image as published on Docker Cloud (aka Docker Hub), we need to publish this image to the **casecommons/elasticsearch** ECR repository that was established when we created [ECR resources]({{< relref "ecr-resources/index.md" >}}), as our ECS container instances can only reach the Internet via the [Web Proxy]({{< relref "web-proxy/index.md" >}}), and the default whitelist on the proxy only allows access to AWS API and service endpoints include the EC2 container registry, but does not permit access to Docker Cloud. 

1\. Pull the Elasticsearch from Docker Cloud to your local environment.

```bash
$ docker pull elasticsearch:2.4-alpine
Using tag: 2.4-alpine
latest: Pulling from elasticsearch
Digest: sha256:7b0d0c8b63db2f02b3c58e451f0a5c4ced9bb5deac66bdc6010d2e07b9e85860
Status: Image is up to date for elasticsearch:2.4-alpine
```

2\. Login to the **demo-resources** ECR service, tag and push the Elasticsearch image with the full ECR repository name.

```bash
$ export AWS_PROFILE=demo-resources-admin
$ eval $(aws ecr get-login)
Enter MFA code: ******
Flag --email has been deprecated, will be removed in 1.14.
Login Succeeded
$ docker tag elasticsearch:2.4-alpine 160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/elasticsearch
$ docker push 160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/elasticsearch
The push refers to a repository [160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/elasticsearch]
442c7116f27b: Pushed
b12e0801104f: Pushed
d97676491a0b: Pushed
0227e2521da2: Pushed
b5179e9e72cd: Pushed
494d9dab3fce: Pushed
6f7515f19096: Pushed
da07d9b32b00: Pushed
7cbcbac42c44: Pushed
latest: digest: sha256:7b0d0c8b63db2f02b3c58e451f0a5c4ced9bb5deac66bdc6010d2e07b9e85860 size: 2199
```

3\. In the AWS console under **ECS > Repositories**, you should now be able to see your newly published image in the **casecommons/elasticsearch** repository:

![Elasticsearch Image](/images/ecr-elastic-image.png)

## Creating the Intake Base Image

The Intake API Docker image requires an Intake base image to be available when building the Intake API Docker image, which means we must publish this image to our new ECR repositories.

1\. Clone the Casecommons Intake Base repository to your local environment:

```bash
$ git clone https://github.com/Casecommons/base_docker_image_for_ca_intake.git
Cloning into 'base_docker_image_for_ca_intake'...
remote: Counting objects: 38, done.
remote: Total 38 (delta 0), reused 0 (delta 0), pack-reused 38
Receiving objects: 100% (38/38), 4.41 KiB | 0 bytes/s, done.
Resolving deltas: 100% (16/16), done.
```

2\. Run the `docker build` command, referencing `Dockerfile.base` and specifying a fully qualified image name of the **intake-base** repository we created in [ECR Resources]({{< relref "ecr-resources/index.md" >}}):

```bash
$ docker build -t 160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/intake-base -f Dockerfile.base .
Sending build context to Docker daemon 9.728 kB
Step 1/5 : FROM ruby:2.4.0-slim
 ---> 6c02426a7abf
Step 2/5 : RUN apt-get update -y &&   apt-get upgrade -y &&   apt-get install -y     jq     curl     python-dev     python-setuptools &&   easy_install pip &&   pip install awscli &&   curl -L -o /usr/bin/confd https://github.com/kelseyhightower/confd/releases/download/v0.12.0-alpha3/confd-0.12.0-alpha3-linux-amd64 &&   chmod +x /usr/bin/confd &&   mkdir -p /etc/confd
 ---> Using cache
 ---> 78eb65e1eecd
Step 3/5 : COPY entrypoint.sh /usr/bin/entrypoint
 ---> Using cache
 ---> 8f9518ae9718
Step 4/5 : ENTRYPOINT /usr/bin/entrypoint
 ---> Using cache
 ---> a23943a21477
Step 5/5 : LABEL application intake_accelerator
 ---> Using cache
 ---> 1c42aa461964
Successfully built 1c42aa461964
```

3\. Run the `docker push` command, referencing the fully qualified image name of the **intake-base** repository:

```bash
$ docker push 160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/intake-base
The push refers to a repository [160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/intake-base]
cc269cc723a3: Pushed
985910e761d0: Pushed
e302f1438372: Pushed
7cebbf491a73: Pushed
ba3d5a81c1e6: Pushed
5d53f93940f5: Pushed
38dcb92de9f2: Pushed
a2ae92ffcd29: Pushed
latest: digest: sha256:e870f9900f5af857c101e39d8683aca8f7dee96e37339a90579f3c857203fa64 size: 1996
```

4\. In the AWS console under **ECS > Repositories**, you should now be able to see your newly published image in the **casecommons/intake-base** repository:

![Intake Base Image](/images/intake-base-image.png)

## Creating the Intake API Image

Now that we have built the Intake base Docker image, we can build and publish the Intake API Docker image to the **casecommons/intake-api** repository, so that it is available to ECS instances when we deploy the CloudFormation stack for running the Intake API application.

1\. Clone the Intake API application to your local environment and ensure you checkout the **workflow_enhancements** branch:

{{< highlight make "hl_lines=9" >}}
$ git clone https://github.com/ca-cwds/intake_api_prototype.git
Cloning into 'intake_api_prototype'...
remote: Counting objects: 2140, done.
remote: Compressing objects: 100% (188/188), done.
remote: Total 2140 (delta 98), reused 2 (delta 2), pack-reused 1931
Receiving objects: 100% (2140/2140), 273.79 KiB | 237.00 KiB/s, done.
Resolving deltas: 100% (1169/1169), done.
$ cd intake_api_prototype
$ git checkout workflow_enhancements
Branch workflow_enhancements set up to track remote branch workflow_enhancements from origin.
Switched to a new branch 'workflow_enhancements'
{{< /highlight >}}

2\. Open the `Makefile` at the root of the **intake_api_prototype** repository and modify the highlighted settings:

{{< highlight make "hl_lines=3 4 5 6" >}}
# Variables
PROJECT_NAME ?= intake_api_prototype
ORG_NAME ?= casecommons
REPO_NAME ?= intake-api
DOCKER_REGISTRY ?= 160775127577.dkr.ecr.us-west-2.amazonaws.com
AWS_ACCOUNT_ID ?= 160775127577
DOCKER_LOGIN_EXPRESSION := eval $$(aws ecr get-login --registry-ids $(AWS_ACCOUNT_ID))
...
...
{{< /highlight >}}

Here we ensure the `Makefile` settings are configured with the correct Docker registry name (`DOCKER_REGISTRY`), organization name (`ORG_NAME`), repository name (`REPO_NAME`) and AWS account ID (`AWS_ACCOUNT_ID`).

3\. Open the `docker/release/Dockerfile` file and modify the `FROM` directive to reference the **intake-base** image you published earlier:

{{< highlight make "hl_lines=1" >}}
FROM 160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/intake-base

ENV APP_HOME /intake_api_prototype
RUN mkdir $APP_HOME
WORKDIR $APP_HOME
...
...
{{< /highlight >}}

4\. Open the `docker/release/docker-compose.yml` file and modify the `image` variable for the `nginx` service to reference the **Nginx** image you published earlier:

{{< highlight python "hl_lines=8" >}}
version: '2'
...
...
services:
...
...
  nginx:
    image: 160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/nginx
    ports:
      - "${HTTP_PORT}"
    environment:
      HTTP_PORT: ${HTTP_PORT}
      WEB_ROOT: /intake_api_prototype/public
      UPSTREAM_URL: unix:///tmp/app.sock
      HEALTHCHECK: curl -fs localhost:${HTTP_PORT}/api/v1/screenings
    volumes:
      - webroot:/intake_api_prototype/public
      - tmp:/tmp
{{< /highlight >}}

4\. Run the `make login` command to login to the ECR repository for the **demo-resources** account:

```bash
$ export AWS_PROFILE=demo-resources-admin
$ make login
=> Logging in to Docker registry ...
Enter MFA code: *****
Login Succeeded
=> Logged in to Docker registry
```

5\. Run the `make test` command, which will run unit tests in a Docker build container:

```bash
$ make test
=> Pulling latest images...
=> Building images...
Building rspec_test
...
...
rspec_test_1     | Finished in 4.75 seconds (files took 2.63 seconds to load)
rspec_test_1     | 34 examples, 0 failures
rspec_test_1     |
intakeapiprototypetest_rspec_test_1 exited with code 0
=> Testing complete
```

6\. Run the `make build` command, which will build a Debian package from the application source and copy it to the local `target` folder:

```bash
$ make build
=> Building images...
Building builder
Step 1/14 : FROM ruby:2.4
...
...
=> Removing existing artifacts...
=> Building application artifacts...
Creating intakeapiprototypetest_builder_1
Attaching to intakeapiprototypetest_builder_1
builder_1        | {:timestamp=>"2017-02-28T07:32:34.402593+0000", :message=>"Created package", :path=>"/build_artefacts/intake-api_20170227225021.acefd33_amd64.deb"}
intakeapiprototypetest_builder_1 exited with code 0
=> Copying application artifacts...
=> Build complete
```

7\. Run the `make release` command, which will build the release image for the Intake API application, start a minimal production-like environment and run acceptance tests.

```bash
$ make release
=> Pulling latest images...
Pulling nginx (160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/nginx:latest)...
latest: Pulling from casecommons/nginx
Digest: sha256:98792f09c0199156ce31236705021e1d81de2ac969e06862d0dea6bce03a79d1
Status: Image is up to date for 160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/nginx:latest
=> Building images...
Building app
...
...
=> Starting application...
Creating intakeapiprototype_app_1
=> Starting nginx...
Creating intakeapiprototype_nginx_1
=> Application is running at http://172.16.154.128:32799/api/v1/screenings
```

8\. Run the `make tag latest` command, which will tag the image with the `latest` tag:

```bash
$ make tag latest
=> Tagging release image with tags latest...
=> Tagging complete
```

9\. Run the `make publish` command, which will publish the image to your ECR repository:

```bash
$ make publish
=> Publishing release image to 160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/intake-api...
The push refers to a repository [160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/intake-api]
65326a832931: Pushed
d634cd8ca5bc: Pushed
708e96026525: Pushed
cc269cc723a3: Pushed
985910e761d0: Pushed
e302f1438372: Pushed
7cebbf491a73: Pushed
ba3d5a81c1e6: Pushed
5d53f93940f5: Pushed
38dcb92de9f2: Pushed
a2ae92ffcd29: Pushed
latest: digest: sha256:04cfa51f3d684b0a20132b7d515322816c5d49efccef4eabb9b98e6e03f1e211 size: 2627
=> Publish complete
```

10\. Run the `make clean` command to clean up the local Docker environment.

```bash
$ make clean
=> Destroying development environment...
...
...
=> Removing dangling images...
Deleted: sha256:cf622bbefa4c89420df5d68f8bb47709374ca149f3b26c7faffe225e9ab32bbf
Deleted: sha256:f4bf2522682e8d5e525bf1de15912b6e53fef2e1a9b82878653cf294455a32c5
=> Clean complete
```

11\. In the AWS console under **ECS > Repositories**, you should now be able to see your newly published image in the **casecommons/intake-api** repository:

![Intake API Image](/images/intake-api-image.png)

## Publishing Lambda Functions

Both the Intake API and Intake Accelerator CloudFormation stacks rely on CloudFormation custom resources to perform the following tasks:

- Secrets Management custom resources - used to decrypt KMS encrypted input parameters provided to the stack to resources that do not support native KMS decryption.
- ECS Task custom resources - used to run deployment actions in the form of ECS tasks.  In the case of Intake API, a number of deployment tasks including running database migrations and reindexing Elasticsearch need to be executed as part of each deployment.

CloudFormation custom resources are backed by AWS Lambda functions that are defined in the CloudFormation stack, and each function needs to specify an S3 bucket URL for downloading the source code of the function in a ZIP format.  The [CloudFormation Resources]({{< relref "cloudformation-resources/index.md" >}}) stack we created earlier includes an S3 bucket specifically designed for storing Lambda functions that support CloudFormation custom resources.

In this section we will publish each of the custom resource Lambda functions listed above to the CloudFormation Lambda S3 bucket, which is named according to the convention `<account-id>-cfn-lambda` or `160775127577-cfn-lambda` in the case of our **demo-resources** account.

1\. Clone the KMS Decryption Lambda function repository to your local environment:

```bash
$ git clone https://github.com/Casecommons/lambda-cfn-kms.git
Cloning into 'lambda-cfn-kms'...
remote: Counting objects: 13, done.
remote: Total 13 (delta 0), reused 0 (delta 0), pack-reused 13
Receiving objects: 100% (13/13), 4.82 KiB | 0 bytes/s, done.
Resolving deltas: 100% (2/2), done.
```

2\. Open the `Makefile` at the root of the **lambda-cfn-repository** repository and modify the highlighted settings:

{{< highlight make "hl_lines=3" >}}
# Parameters
FUNCTION_NAME ?= cfnKmsDecrypt
S3_BUCKET ?= 160775127577-cfn-lambda
AWS_DEFAULT_REGION ?= us-west-2
...
...
{{< /highlight >}}

Here we ensure the S3 bucket the Lambda function will be published to is the correct bucket for the **demo-resources** account (`S3_BUCKET`).

3\. Run the `make publish` command, which will build a ZIP archive of your Lambda function and upload it to the S3 bucket:

{{< highlight bash "hl_lines=29" >}}
$ export AWS_PROFILE=demo-resources-admin
$ make publish
=> Removed all distributions
=> Building cfnKmsDecrypt.zip...
Collecting cfn-lambda-handler (from -r requirements.txt (line 1))
  Using cached cfn_lambda_handler-1.0.2-py2.py3-none-any.whl
Installing collected packages: cfn-lambda-handler
Successfully installed cfn-lambda-handler-1.0.2
  adding: cfn_kms_decrypt.py (deflated 54%)
  adding: requirements.txt (stored 0%)
  adding: setup.cfg (stored 0%)
  adding: vendor/ (stored 0%)
  adding: vendor/cfn_lambda_handler/ (stored 0%)
  adding: vendor/cfn_lambda_handler/__init__.py (deflated 28%)
  adding: vendor/cfn_lambda_handler/cfn_lambda_handler.py (deflated 67%)
  adding: vendor/cfn_lambda_handler-1.0.2.dist-info/ (stored 0%)
  adding: vendor/cfn_lambda_handler-1.0.2.dist-info/DESCRIPTION.rst (stored 0%)
  adding: vendor/cfn_lambda_handler-1.0.2.dist-info/INSTALLER (stored 0%)
  adding: vendor/cfn_lambda_handler-1.0.2.dist-info/METADATA (deflated 56%)
  adding: vendor/cfn_lambda_handler-1.0.2.dist-info/metadata.json (deflated 52%)
  adding: vendor/cfn_lambda_handler-1.0.2.dist-info/RECORD (deflated 49%)
  adding: vendor/cfn_lambda_handler-1.0.2.dist-info/top_level.txt (stored 0%)
  adding: vendor/cfn_lambda_handler-1.0.2.dist-info/WHEEL (deflated 14%)
=> Built build/cfnKmsDecrypt.zip
=> Publishing cfnKmsDecrypt.zip to s3://160775127577-cfn-lambda...
Enter MFA token: ******

=> Published to S3 URL: https://s3-us-west-2.amazonaws.com/160775127577-cfn-lambda/cfnKmsDecrypt.zip
=> S3 Object Version: BIUpXhvw6w3f1LTctGVZxOpFD38lCHH1
{{< /highlight >}}

Take note of the S3 object version, as we need to provide this as an input to any CloudFormation stacks that will use this function.

4\. Next we will clone the ECS Task Runner Lambda function repository to your local environment:

```bash
$ git clone https://github.com/Casecommons/lambda-ecs-tasks.git
Cloning into 'lambda-ecs-tasks'...
remote: Counting objects: 20, done.
remote: Total 20 (delta 0), reused 0 (delta 0), pack-reused 20
Receiving objects: 100% (20/20), 513.86 KiB | 423.00 KiB/s, done.
```

5\. Open the `Makefile` at the root of the **lambda-ecs-tasks** repository and modify the highlighted settings:

{{< highlight make "hl_lines=6" >}}
# Project variables
export PROJECT_NAME ?= lambda-ecs-tasks

# Parameters
export FUNCTION_NAME ?= cfnEcsTasks
S3_BUCKET ?= 160775127577-cfn-lambda
AWS_DEFAULT_REGION ?= us-west-2
...
...
{{< /highlight >}}

Here we ensure the S3 bucket the Lambda function will be published to is the correct bucket for the **demo-resources** account (`S3_BUCKET`).

6\. Run the `make build` and `make publish` commands, which will build a ZIP archive of your Lambda function and upload it to the S3 bucket:

{{< highlight bash "hl_lines=15" >}}
$ export AWS_PROFILE=demo-resources-admin
$ make build
=> Building cfnEcsTasks.zip...
Collecting voluptuous (from -r requirements.txt (line 1))
Collecting cfn-lambda-handler (from -r requirements.txt (line 2))
  Using cached cfn_lambda_handler-1.0.2-py2.py3-none-any.whl
...
...
  adding: vendor/setuptools-34.3.0.dist-info/WHEEL (deflated 14%)
  adding: vendor/setuptools-34.3.0.dist-info/zip-safe (stored 0%)
=> Built build/cfnEcsTasks.zip
$ make publish
=> Publishing cfnEcsTasks.zip to s3://160775127577-cfn-lambda...
=> Published to S3 URL: https://s3.amazonaws.com/160775127577-cfn-lambda/cfnEcsTasks.zip
=> S3 Object Version: MiOiZi.9.AHyYQcFibBAvWYFD4xFJJ1c
{{< /highlight >}}

Take note of the S3 object version, as we need to provide this as an input to any CloudFormation stacks that will use this function.

7\. In the AWS console, navigate to the S3 dashboard and open the S3 bucket **\<account-id\>-cfn-lambda**.  Notice that each Lambda function has been uploaded as a ZIP file:

![S3 Dashboard](/images/s3-dashboard.png)
![Lambda Bucket](/images/lambda-bucket.png)

## Installing the Playbook

We now have all the supporting pieces in place to deploy the Intake API application stack to the **demo resources** account.  Instead of creating a new playbook as we have done previously in this tutorial, we will instead clone an existing playbook and add a new environment called **demo-resources** to the playbook.

1\. Clone the [Intake API AWS project](https://github.com/casecommons/intake-api-aws) to your local environment.

```bash
$ git clone https://github.com/Casecommons/intake-api-aws.git
  Cloning into 'intake-api-aws'...
  remote: Counting objects: 58, done.
  remote: Compressing objects: 100% (6/6), done.
  remote: Total 58 (delta 0), reused 0 (delta 0), pack-reused 49
  Receiving objects: 100% (58/58), 23.82 KiB | 0 bytes/s, done.
  Resolving deltas: 100% (18/18), done.
$ cd intake-api-aws
```

3\.  Install the required Ansible roles for the playbook using the `ansible-galaxy` command as demonstrated below:

{{< highlight python >}}
$ ansible-galaxy install -r roles/requirements.yml --force
- extracting aws-cloudformation to /Users/jmenga/Source/casecommons/intake-api-aws/roles/aws-cloudformation
- aws-cloudformation was installed successfully
- extracting aws-sts to /Users/jmenga/Source/casecommons/intake-api-aws/roles/aws-sts
- aws-sts was installed successfully
{{< /highlight >}}

5\.  Review the `group_vars/all/vars.yml` file, which contains global settings for the playbook:

{{< highlight python >}}
cf_stack_name: "{{ 'intake-api-' + env }}"
cf_stack_tags:
  org:business:owner: CA Intake
  org:business:product: Intake API
  org:business:severity: High
  org:tech:environment: "{{ env }}"
  org:tech:contact: pema@casecommons.org

cf_stack_inputs:
  ApplicationAutoscalingDesiredCount: "{{ config_application_desired_count }}"
  ApplicationAMI: "{{ config_application_ami }}"
  ApplicationInstanceType: "{{ config_application_instance_type }}"
  ApplicationLoadBalancerPort: "{{ config_application_frontend_port }}"
  ApplicationPort: "{{ config_application_port }}"
  ApplicationKeyName: "{{ config_application_keyname }}"
  ApplicationDockerImage: "{{ config_application_image }}"
  ApplicationDockerImageTag: "{{ config_application_tag }}"
  ApplicationDomain: "{{ config_application_domain }}"
  ElasticsearchAutoscalingDesiredCount: "{{ config_es_desired_count }}"
  ElasticsearchAMI: "{{ config_es_ami }}"
  ElasticsearchInstanceType: "{{ config_es_instance_type }}"
  ElasticsearchLoadBalancerPort: "{{ config_es_frontend_port }}"
  ElasticsearchPort: "{{ config_es_port }}"
  ElasticsearchKeyName: "{{ config_es_keyname }}"
  ElasticsearchDockerImage: "{{ config_es_image }}"
  ElasticsearchDockerImageTag: "{{ config_es_tag }}"
  Environment: "{{ env }}"
  LogRetention: "{{ config_log_retention }}"
  NginxDockerImage: "{{ config_nginx_image }}"
  NginxDockerImageTag: "{{ config_nginx_tag }}"
  DbStorage: "{{ config_db_storage }}"
  DbInstanceType: "{{ config_db_instance_type }}"
  DbName: "{{ config_db_name }}"
  DbUsername: "{{ config_db_username }}"
  DbPassword: "{{ config_db_password }}"
  SecretKeyBaseCipher: "{{ config_application_secret_key_base }}"
  LambdaCfnKmsDecryptVersion: "{{ config_cfn_lambda_kms_decrypt_version }}"
  LambdaCfnEcsTasksVersion: "{{ config_cfn_lambda_ecs_tasks_version }}"
{{< /highlight >}}

You can see this stack has many different stack inputs, which revolve around application settings, elasticsearch settings and database settings.

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

4\. Copy the existing settings from `group_vars/dev/vars.yml` to `group_vars/demo/vars.yml` and modify as shown below:

{{< highlight ini "hl_lines=2 5 8 9 13 14 17 21 22 25 28 29 41 48 49" >}}
# STS settings
sts_role_arn: arn:aws:iam::160775127577:role/admin

# Application settings
config_application_keyname: admin
config_application_instance_type: t2.micro
config_application_desired_count: 2
config_application_ami: ami-4662e126
config_application_image: 160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/intake-api
config_application_tag: latest
config_application_frontend_port: 443
config_application_port: 3000
config_application_secret_key_base: xxxxx
config_application_domain: demo.cloudhotspot.co

# Nginx settings
config_nginx_image: 160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/nginx
config_nginx_tag: latest

# Load balancer settings
config_lb_certificate_arn:
  Fn::ImportValue: DemoCloudhotspotCoCertificateArn

# Elasticsearch settings
config_es_keyname: admin
config_es_instance_type: t2.micro
config_es_desired_count: 1
config_es_ami: ami-4662e126
config_es_image: 160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/elasticsearch
config_es_tag: latest
config_es_frontend_port: 9200
config_es_port: 9200

# Database settings
config_db_multi_az: false
config_db_deletion_policy: Delete
config_db_storage: 10
config_db_instance_type: db.t2.micro
config_db_name: casebook_api_production
config_db_username: casebook
config_db_password: xxxxx

# Log settings
config_log_retention: 30
config_log_deletion_policy: Delete

# CloudFormation custom resource settings 
config_cfn_lambda_kms_decrypt_version: "BIUpXhvw6w3f1LTctGVZxOpFD38lCHH1"
config_cfn_lambda_ecs_tasks_version: "MiOiZi.9.AHyYQcFibBAvWYFD4xFJJ1c"
{{< /highlight >}}

Here we target the **demo-resources** account by specifying the **demo-resources** IAM **admin** role in the `sts_role_arn` variable, whilst the remaining settings configure the Intake APi application stack specify to the **demo-resources** template account:

- `config_application_keyname` - specifies the name of the EC2 key pair that ECS container instances will be created with.  Notice this matches the name of the [EC2 key pair created earlier]({{< relref "web-proxy/index.md#creating-an-ec2-key-pair" >}}). 
- `config_application_ami` - specifies the AMI ID of the image used to create the ECS container instances.  Notice this matches the ID of the [AMI created earlier]({{< relref "web-proxy/index.md#creating-an-ecs-ami" >}})
- `config_application_image` - specifies the Docker image used to run the Intake API application containers.  Notice this matches the [image we created earlier]({{< relref "intake-api/index.md#creating-the-intake-api-image" >}})
- `config_application_secret_key_base` - an encrypted application setting that provides cryptographic material for an application secret key.  We will securely generate an encrypted value for this setting shortly.
- `config_application_domain` - specifies the base domain that our application will be served from.
- `config_nginx_image` - specifies the Docker image used to run the Intake API Nginx containers.  Notice this matches the [image we created earlier]({{< relref "intake-api/index.md#creating-an-nginx-image" >}})
- `config_lb_certificate_arn` - specifies the ARN of the AWS certificate manager (ACM) certificate to serve from the application load balancer for HTTPS connections.  Notice that we can specify this setting as a CloudFormation instrinsic function, as the template is configured to cast the configured value to a JSON object.  The intrinsic function imports the CloudFormation export `DemoCloudhotspotCoCertificateArn`, which was created earlier when we [created the certificate]({{< relref "intake-api/index.md#creating-a-certificate" >}}) in the security resources playbook.
- `config_es_keyname` - specifies the name of the EC2 key pair that ECS container instances will be created with. 
- `config_es_ami` - specifies the AMI ID of the image used to create the ECS container instances.  
- `config_es_image` - specifies the Docker image used to run the Intake API Elasticsearch containers.  Notice this matches the [image we created earlier]({{< relref "intake-api/index.md#creating-an-elasticsearch-image" >}})
- `config_db_password` - the encrypted password for the Intake API database.  We will securely generate an encrypted value for this setting shortly.
- `config_cfn_lambda_kms_decrypt_version` - the S3 object version of the KMS decrypt Lambda function used for KMS decrypt custom resources.  Notice the object version matches [the version we published earlier]({{< relref "intake-api/index.md#creating-lambda-functions" >}}).
- `config_cfn_lambda_ecs_tasks_version` - the S3 object version of the ECS tasks Lambda function used for ECS task custom resources.  Notice the object version matches [the version we published earlier]({{< relref "intake-api/index.md#creating-lambda-functions" >}}).

## Creating Secrets using KMS

Our **demo** environment settings include two settings that we need to generate encrypted values for.

The approach here is to simply use the AWS Key Management Service (KMS) to encrypt the plaintext value of each secret.  We will use the KMS key that was created as part of the [CloudFormation resources]({{< relref "cloudformation-resources/index.md" >}}) stack.

1\. In the AWS console, open the **CloudFormation > Exports** from the CloudFormation dashboard.  Copy the **CfnMasterKey** export value, which defines the key ID of the KMS key we will use for encryption.

![KMS Master Key](/images/cfn-master-key.png)

2\. In a local shell configured with an admin profile targetting the **demo resources** account, run the following command to generate ciphertext for the `config_application_secret_key_base` variable:

```bash
$ export AWS_PROFILE=demo-resources-admin
$ aws kms encrypt --key-id 11710b57-c424-4df7-ab3d-20358820edd9 --plaintext $(openssl rand -base64 32)
Enter MFA code: ******
'{
    "KeyId": "arn:aws:kms:us-west-2:160775127577:key/11710b57-c424-4df7-ab3d-20358820edd9",
    "CiphertextBlob": "AQECAHjbSbOZ8FLk7XffvdtrDewDyQKH9bOaMrY6jf+N3si+SQAAAIswgYgGCSqGSIb3DQEHBqB7MHkCAQAwdAYJKoZIhvcNAQcBMB4GCWCGSAFlAwQBLjARBAylerEZFMHMqvVgDiICARCAR36t/jpVODBIevAAeKEbHH+Tibj2/buhzmeoSiN+fm6T6CL1AE+yjRZ7B7FXp5S2RhYEwBrWLGD26r3NudZ9aj0IvLI8BwPS"
}
```

3\. Copy the `CiphertextBlob` value from the `aws kms encrypt` command output and set the `config_application_secret_key_base` variable in `group_vars/demo/vars.yml` to this value:

{{< highlight ini "hl_lines=13" >}}
# STS settings
sts_role_arn: arn:aws:iam::160775127577:role/admin

# Application settings
config_application_keyname: admin
config_application_instance_type: t2.micro
config_application_desired_count: 2
config_application_ami: ami-4662e126
config_application_image: 160775127577.dkr.ecr.us-west-2.amazonaws.com/casecommons/intake-api
config_application_tag: latest
config_application_frontend_port: 443
config_application_port: 3000
config_application_secret_key_base: AQECAHjbSbOZ8FLk7XffvdtrDewDyQKH9bOaMrY6jf+N3si+SQAAAIswgYgGCSqGSIb3DQEHBqB7MHkCAQAwdAYJKoZIhvcNAQcBMB4GCWCGSAFlAwQBLjARBAylerEZFMHMqvVgDiICARCAR36t/jpVODBIevAAeKEbHH+Tibj2/buhzmeoSiN+fm6T6CL1AE+yjRZ7B7FXp5S2RhYEwBrWLGD26r3NudZ9aj0IvLI8BwPS
config_application_domain: demo.cloudhotspot.co
...
...
{{< /highlight >}}

5\. Repeat the `aws kms encryt` command to generate ciphertext for the `config_db_password` variable:

```bash
$ export AWS_PROFILE=demo-resources-admin
$ aws kms encrypt --key-id 11710b57-c424-4df7-ab3d-20358820edd9 --plaintext "supersecretpassword"
Enter MFA code: ******
'{
    "KeyId": "arn:aws:kms:us-west-2:160775127577:key/11710b57-c424-4df7-ab3d-20358820edd9",
    "CiphertextBlob": "AQECAHjbSbOZ8FLk7XffvdtrDewDyQKH9bOaMrY6jf+N3si+SQAAAHEwbwYJKoZIhvcNAQcGoGIwYAIBADBbBgkqhkiG9w0BBwEwHgYJYIZIAWUDBAEuMBEEDLRuXWaxRLSxR5//zAIBEIAuindRhw7U4ERU7xSWH/5QX8lJ1F2cZmjHCJh6EFTkD/5iU7BGXs/PVo1iy8czgw=="
}
```

6\. Copy the `CiphertextBlob` value from the `aws kms encrypt` command output and set the `config_application_secret_key_base` variable in `group_vars/demo/vars.yml` to this value:

{{< highlight ini "hl_lines=11" >}}
# STS settings
sts_role_arn: arn:aws:iam::160775127577:role/admin
...
...
# Database settings
config_db_multi_az: false
config_db_deletion_policy: Delete
config_db_storage: 10
config_db_instance_type: db.t2.micro
config_db_name: casebook_api_production
config_db_username: AQECAHjbSbOZ8FLk7XffvdtrDewDyQKH9bOaMrY6jf+N3si+SQAAAHEwbwYJKoZIhvcNAQcGoGIwYAIBADBbBgkqhkiG9w0BBwEwHgYJYIZIAWUDBAEuMBEEDLRuXWaxRLSxR5//zAIBEIAuindRhw7U4ERU7xSWH/5QX8lJ1F2cZmjHCJh6EFTkD/5iU7BGXs/PVo1iy8czgw==
...
...
{{< /highlight >}}

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

3\.  The playbook will take approxiately 15 minutes to create the CloudFormation stack and associated resources.  Whilst the CloudFormation stack is being created, you can review the CloudFormation stack that was generated in the `build/<timestamp>` folder:

```bash
$ tree build
build
└── 20170301192616
    ├── intake-api-demo-config.json
    ├── intake-api-demo-policy.json
    ├── intake-api-demo-stack.json
    └── intake-api-demo-stack.yml
```

The following shows the `intake-api-demo-stack.yml` file that was generated and uploaded to CloudFormation:

```python
AWSTemplateFormatVersion: "2010-09-09"

Description: Intake API - demo

Parameters:
  ApplicationAMI:
    Type: String
    Description: Application Amazon Machine Image ID
  ApplicationInstanceType:
    Type: String
    Description: Application EC2 Instance Type
  ApplicationAutoscalingDesiredCount:
    Type: Number
    Description: Application AutoScaling Group Desired Count
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
  ApplicationLoadBalancerPort:
    Type: Number
    Description: Application Front End HTTP Port
  ApplicationPort:
    Type: Number
    Description: Application HTTP Port
  ApplicationDomain:
    Type: String
    Description: Base public domain of the application URL
  ElasticsearchAMI:
    Type: String
    Description: Elasticsearch Amazon Machine Image ID
  ElasticsearchInstanceType:
    Type: String
    Description: Elasticsearch EC2 Instance Type
  ElasticsearchAutoscalingDesiredCount:
    Type: Number
    Description: Elasticsearch AutoScaling Group Desired Count
  ElasticsearchDockerImage:
    Type: String
    Description: Docker Image for Elasticsearch
  ElasticsearchDockerImageTag:
    Type: String
    Description: Docker Image Tag for Elasticsearch
    Default: latest
  ElasticsearchKeyName:
    Type: String
    Description: EC2 Key Pair for Elasticsearch SSH Access
  ElasticsearchLoadBalancerPort:
    Type: Number
    Description: Elasticsearch Front End HTTP Port
  ElasticsearchPort:
    Type: Number
    Description: Elasticsearch Port
  Environment:
    Type: String
    Description: Stack Environment
  LogRetention:
    Type: Number
    Description: Log retention in days
  NginxDockerImage:
    Type: String
    Description: Docker Image for Nginx
  NginxDockerImageTag:
    Type: String
    Description: Docker Image Tag for Nginx
    Default: latest
  DbStorage:
    Type: Number
    Description: Database Allocated Storage
  DbInstanceType:
    Type: String
    Description: Database Instance Type
  DbName:
    Type: String
    Description: Database Name
  DbUsername:
    Type: String
    Description: Database Username
  DbPassword:
    Type: String
    NoEcho: "True"
    Description: Encrypted Database Password
  SecretKeyBaseCipher:
    Type: String
    Description: KMS Encrypted Secret Key Base
  LambdaCfnKmsDecryptVersion:
    Type: String
    Description: S3 Object Version of CloudFormation KMS Decrypt function
  LambdaCfnEcsTasksVersion:
    Type: String
    Description: S3 Object Version of CloudFormation ECS Tasks function

Resources:
  ApplicationDnsRecord:
    Type: "AWS::Route53::RecordSet"
    Properties:
      Name: 
        Fn::Sub: "${Environment}-intake-api.${ApplicationDomain}"
      TTL: "300"
      HostedZoneName: 
        Fn::Sub: "${ApplicationDomain}."
      Type: "CNAME"
      Comment: 
        Fn::Sub: "Intake API Application - ${Environment}"
      ResourceRecords: 
        - Fn::Sub: "${ApplicationLoadBalancer.DNSName}"
  ApplicationLoadBalancer:
    Type: "AWS::ElasticLoadBalancingV2::LoadBalancer"
    Properties:
      Scheme: "internet-facing"
      SecurityGroups:
       - Ref: "ApplicationLoadBalancerSecurityGroup"
      Subnets:
        - Fn::ImportValue: DefaultPublicSubnetA
        - Fn::ImportValue: DefaultPublicSubnetB
      LoadBalancerAttributes: 
        - Key: "deletion_protection.enabled"
          Value: false
        - Key: "idle_timeout.timeout_seconds"
          Value: 30
      Tags:
        - Key: "Name"
          Value:
            Fn::Sub: ${AWS::StackName}-${Environment}-lb
  ApplicationLoadBalancerApplicationListener:
    Type: "AWS::ElasticLoadBalancingV2::Listener"
    Properties:
      Certificates:
        - CertificateArn: {"Fn::ImportValue": "DemoCloudhotspotCoCertificateArn"}
      DefaultActions:
        - TargetGroupArn: { "Ref": "IntakeApiServiceTargetGroup" }
          Type: forward
      LoadBalancerArn: { "Ref": "ApplicationLoadBalancer" }
      Port: { "Ref": "ApplicationLoadBalancerPort" }
      Protocol: "HTTPS"
  IntakeApiServiceTargetGroup:
    Type: "AWS::ElasticLoadBalancingV2::TargetGroup"
    Properties:
      VpcId:
        Fn::ImportValue: "DefaultVpcId"
      Protocol: "HTTP"
      Port: { "Ref": "ApplicationPort" }
      HealthCheckPath: "/api/v1/screenings"
      TargetGroupAttributes:
        - Key: "deregistration_delay.timeout_seconds"
          Value: 60
  ApplicationLoadBalancerSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      VpcId:
        Fn::ImportValue: "DefaultVpcId"
      GroupDescription: "Intake API Load Balancer Security Group"
      SecurityGroupIngress:
        - IpProtocol: "tcp"
          FromPort: { "Ref": "ApplicationLoadBalancerPort" }
          ToPort: { "Ref": "ApplicationLoadBalancerPort" }
          CidrIp: "0.0.0.0/0"
  ApplicationLoadBalancerToApplicationIngress:
    Type: "AWS::EC2::SecurityGroupIngress"
    Properties:
      IpProtocol: "tcp"
      FromPort: { "Ref": "ApplicationPort" }
      ToPort: { "Ref": "ApplicationPort" }
      GroupId: { "Ref": "ApplicationAutoscalingSecurityGroup" }
      SourceSecurityGroupId: { "Ref": "ApplicationLoadBalancerSecurityGroup" }
  ApplicationLoadBalancerToApplicationEgress:
    Type: "AWS::EC2::SecurityGroupEgress"
    Properties:
      IpProtocol: "tcp"
      FromPort: { "Ref": "ApplicationPort" }
      ToPort: { "Ref": "ApplicationPort" }
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
        - Fn::ImportValue: "DefaultMediumSubnetA"
        - Fn::ImportValue: "DefaultMediumSubnetB"
      LaunchConfigurationName: { "Ref" : "ApplicationAutoscalingLaunchConfiguration" }
      MinSize: "0"
      MaxSize: "4"
      DesiredCapacity: { "Ref": "ApplicationAutoscalingDesiredCount"}
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
                DOCKER_NETWORK_MODE: host
                PROXY_URL:
                  Fn::ImportValue: DefaultProxyURL
              cwd: "/home/ec2-user/"
          files:
            /etc/ecs/ecs.config:
              content: 
                Fn::Sub: "ECS_CLUSTER=${ApplicationCluster}\n"
    Properties:
      ImageId: { "Ref": "ApplicationAMI" }
      InstanceType: { "Ref": "ApplicationInstanceType" }
      IamInstanceProfile: { "Ref": "ApplicationAutoscalingInstanceProfile" }
      KeyName: { "Ref": "ApplicationKeyName" }
      SecurityGroups:
        - Ref: "ApplicationAutoscalingSecurityGroup"
      UserData: 
        Fn::Base64:
          Fn::Join: ["\n", [
            "#!/bin/bash",
            "set -e",
            "Fn::Join": ["", [
              "Fn::Sub": "/opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource ApplicationAutoscalingLaunchConfiguration --region ${AWS::Region}",
              "    --http-proxy ", "Fn::ImportValue": "DefaultProxyURL", 
              "    --https-proxy ", "Fn::ImportValue": "DefaultProxyURL"
            ] ],
            "Fn::Join": ["", [
              "Fn::Sub": "/opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource ApplicationAutoscaling --region ${AWS::Region}",
              "    --http-proxy ", "Fn::ImportValue": "DefaultProxyURL", 
              "    --https-proxy ", "Fn::ImportValue": "DefaultProxyURL"
            ] ]
          ] ]
  ApplicationAutoscalingSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      VpcId:
        Fn::ImportValue: "DefaultVpcId"
      GroupDescription: "Intake API Security Group"
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
          FromPort: 3128
          ToPort: 3128
          DestinationSecurityGroupId:
            Fn::ImportValue: DefaultProxySecurityGroup
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
  ApplicationToElasticsearchLoadBalancerIngress:
    Type: "AWS::EC2::SecurityGroupIngress"
    Properties:
      IpProtocol: "tcp"
      FromPort: { "Ref": "ElasticsearchLoadBalancerPort"}
      ToPort: { "Ref": "ElasticsearchLoadBalancerPort"}
      GroupId: { "Ref": "ElasticsearchLoadBalancerSecurityGroup" }
      SourceSecurityGroupId: { "Ref": "ApplicationAutoscalingSecurityGroup" }
  ApplicationToElasticsearchLoadBalancerEgress:
    Type: "AWS::EC2::SecurityGroupEgress"
    Properties:
      IpProtocol: "tcp"
      FromPort: { "Ref": "ElasticsearchLoadBalancerPort"}
      ToPort: { "Ref": "ElasticsearchLoadBalancerPort"}
      GroupId: { "Ref": "ApplicationAutoscalingSecurityGroup" }
      DestinationSecurityGroupId: { "Ref": "ElasticsearchLoadBalancerSecurityGroup" }
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
        - {"PolicyName": "CloudWatchLogs", "PolicyDocument": {"Version": "2012-10-17", "Statement": [{"Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents", "logs:DescribeLogStreams"], "Resource": {"Fn::Join": [":", ["arn:aws:logs", {"Ref": "AWS::Region"}, {"Ref": "AWS::AccountId"}, "log-group", "intake-api-demo*"]]}, "Effect": "Allow"}]}}
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
              - Effect: "Allow"
                Action:
                - "kms:Decrypt"
                - "kms:DescribeKey"
                Resource:
                  Fn::ImportValue: "CfnMasterKeyArn"
  ApplicationAutoscalingInstanceProfile:
    Type: "AWS::IAM::InstanceProfile"
    Properties:
      Path: "/"
      Roles: [ { "Ref": "ApplicationAutoscalingRole" } ]
  ElasticsearchDnsRecord:
    Type: "AWS::Route53::RecordSet"
    Properties:
      Name:
        Fn::Join: ["", [
          "Fn::Sub": "${Environment}-intake-api-es.",
          "Fn::ImportValue": "DefaultVpcDomain"
        ] ]
      TTL: "300"
      HostedZoneName:
        Fn::ImportValue: DefaultVpcZone
      Type: "CNAME"
      Comment: 
        Fn::Sub: "Intake API Elasticsearch - ${Environment}"
      ResourceRecords: 
        - Fn::Sub: "${ElasticsearchLoadBalancer.DNSName}"
  ElasticsearchLoadBalancer:
    Type: "AWS::ElasticLoadBalancingV2::LoadBalancer"
    Properties:
      Scheme: "internal"
      SecurityGroups:
       - Ref: "ElasticsearchLoadBalancerSecurityGroup"
      Subnets:
        - Fn::ImportValue: DefaultHighSubnetA
        - Fn::ImportValue: DefaultHighSubnetB
      LoadBalancerAttributes: 
        - Key: "deletion_protection.enabled"
          Value: false
        - Key: "idle_timeout.timeout_seconds"
          Value: 30
      Tags:
        - Key: "Name"
          Value:
            Fn::Sub: ${AWS::StackName}-${Environment}-elasticsearch-lb
  ElasticsearchLoadBalancerListener:
    Type: "AWS::ElasticLoadBalancingV2::Listener"
    Properties:
      DefaultActions:
        - TargetGroupArn: { "Ref": "ElasticsearchServiceTargetGroup" }
          Type: forward
      LoadBalancerArn: { "Ref": "ElasticsearchLoadBalancer" }
      Port: { "Ref": "ElasticsearchLoadBalancerPort" }
      Protocol: "HTTP"
  ElasticsearchServiceTargetGroup:
    Type: "AWS::ElasticLoadBalancingV2::TargetGroup"
    Properties:
      VpcId:
        Fn::ImportValue: "DefaultVpcId"
      Protocol: "HTTP"
      Port: { "Ref": "ElasticsearchPort" }
      TargetGroupAttributes:
        - Key: "deregistration_delay.timeout_seconds"
          Value: 60
  ElasticsearchLoadBalancerSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      VpcId:
        Fn::ImportValue: "DefaultVpcId"
      GroupDescription: "Intake API Elasticsearch Load Balancer Security Group"
  ElasticsearchLoadBalancerToElasticsearchIngress:
    Type: "AWS::EC2::SecurityGroupIngress"
    Properties:
      IpProtocol: "tcp"
      FromPort: { "Ref": "ElasticsearchPort" }
      ToPort: { "Ref": "ElasticsearchPort" }
      GroupId: { "Ref": "ElasticsearchAutoscalingSecurityGroup" }
      SourceSecurityGroupId: { "Ref": "ElasticsearchLoadBalancerSecurityGroup" }
  ElasticsearchLoadBalancerToElasticsearchEgress:
    Type: "AWS::EC2::SecurityGroupEgress"
    Properties:
      IpProtocol: "tcp"
      FromPort: { "Ref": "ElasticsearchPort" }
      ToPort: { "Ref": "ElasticsearchPort" }
      GroupId: { "Ref": "ElasticsearchLoadBalancerSecurityGroup" }
      DestinationSecurityGroupId: { "Ref": "ElasticsearchAutoscalingSecurityGroup" }
  ElasticsearchAutoscaling:
    Type: "AWS::AutoScaling::AutoScalingGroup"
    DependsOn:
      - ElasticsearchDmesgLogGroup
      - ElasticsearchDockerLogGroup
      - ElasticsearchEcsAgentLogGroup
      - ElasticsearchEcsInitLogGroup
      - ElasticsearchMessagesLogGroup
    CreationPolicy:
      ResourceSignal:
        Count: { "Ref": "ElasticsearchAutoscalingDesiredCount"}
        Timeout: "PT15M"
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: { "Ref": "ElasticsearchAutoscalingDesiredCount"}
        MinSuccessfulInstancesPercent: "100"
        WaitOnResourceSignals: "true"
        PauseTime: "PT15M"
    Properties:
      VPCZoneIdentifier:
        - Fn::ImportValue: "DefaultHighSubnetA"
        - Fn::ImportValue: "DefaultHighSubnetB"
      LaunchConfigurationName: { "Ref" : "ElasticsearchAutoscalingLaunchConfiguration" }
      MinSize: "0"
      MaxSize: "4"
      DesiredCapacity: { "Ref": "ElasticsearchAutoscalingDesiredCount"}
      Tags:
        - Key: Name
          Value:
            Fn::Sub: ${AWS::StackName}-ElasticsearchAutoscaling-instance
          PropagateAtLaunch: "true"
  ElasticsearchAutoscalingLaunchConfiguration:
    Type: "AWS::AutoScaling::LaunchConfiguration"
    Metadata:
      AWS::CloudFormation::Init:
        config:
          commands:
            10_first_run:
              command: "sh firstrun.sh"
              env:
                STACK_NAME: { "Ref": "AWS::StackName" }
                AUTOSCALING_GROUP: "ElasticsearchAutoscaling"
                AWS_DEFAULT_REGION: { "Ref": "AWS::Region" }
                ECS_CLUSTER: { "Ref": "ElasticsearchCluster" }
                DOCKER_NETWORK_MODE: host
                PROXY_URL:
                  Fn::ImportValue: DefaultProxyURL
              cwd: "/home/ec2-user/"
          files:
            /etc/ecs/ecs.config:
              content: 
                Fn::Sub: "ECS_CLUSTER=${ElasticsearchCluster}\n"
    Properties:
      ImageId: { "Ref": "ElasticsearchAMI" }
      InstanceType: { "Ref": "ElasticsearchInstanceType" }
      IamInstanceProfile: { "Ref": "ElasticsearchAutoscalingInstanceProfile" }
      KeyName: { "Ref": "ElasticsearchKeyName" }
      SecurityGroups:
        - Ref: "ElasticsearchAutoscalingSecurityGroup"
      UserData: 
        Fn::Base64:
          Fn::Join: ["\n", [
            "#!/bin/bash",
            "set -e",
            "Fn::Join": ["", [
              "Fn::Sub": "/opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource ElasticsearchAutoscalingLaunchConfiguration --region ${AWS::Region}",
              "    --http-proxy ", "Fn::ImportValue": "DefaultProxyURL", 
              "    --https-proxy ", "Fn::ImportValue": "DefaultProxyURL"
            ] ],
            "Fn::Join": ["", [
              "Fn::Sub": "/opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource ElasticsearchAutoscaling --region ${AWS::Region}",
              "    --http-proxy ", "Fn::ImportValue": "DefaultProxyURL", 
              "    --https-proxy ", "Fn::ImportValue": "DefaultProxyURL"
            ] ]
          ] ]
  ElasticsearchAutoscalingSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      VpcId:
        Fn::ImportValue: "DefaultVpcId"
      GroupDescription: "Elasticsearch Security Group"
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
          FromPort: 3128
          ToPort: 3128
          DestinationSecurityGroupId:
            Fn::ImportValue: DefaultProxySecurityGroup
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
  ElasticsearchAutoscalingRole:
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
        - {"PolicyName": "CloudWatchLogs", "PolicyDocument": {"Version": "2012-10-17", "Statement": [{"Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents", "logs:DescribeLogStreams"], "Resource": {"Fn::Join": [":", ["arn:aws:logs", {"Ref": "AWS::Region"}, {"Ref": "AWS::AccountId"}, "log-group", "intake-api-demo*"]]}, "Effect": "Allow"}]}}
        - PolicyName: "EC2ContainerInstancePolicy"
          PolicyDocument:
            Statement:
              - Effect: "Allow"
                Action:
                  - "ecs:RegisterContainerInstance"
                  - "ecs:DeregisterContainerInstance"
                Resource: 
                  Fn::Sub: "arn:aws:ecs:${AWS::Region}:${AWS::AccountId}:cluster/${ElasticsearchCluster}"
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
  ElasticsearchAutoscalingInstanceProfile:
    Type: "AWS::IAM::InstanceProfile"
    Properties:
      Path: "/"
      Roles: [ { "Ref": "ElasticsearchAutoscalingRole" } ]
  ApplicationDatabase:
    Type: "AWS::RDS::DBInstance"
    DeletionPolicy: "Delete"
    Properties:
      AllocatedStorage: { "Ref": "DbStorage" }
      AutoMinorVersionUpgrade: "true"
      BackupRetentionPeriod: "7"
      PreferredBackupWindow: "9:00-11:00"
      PreferredMaintenanceWindow: "sun:11:30-sun:13:00"
      StorageType: "gp2"
      DBInstanceClass: { "Ref": "DbInstanceType" }
      Engine: "postgres"
      EngineVersion: "9.5"
      MasterUsername: { "Ref": "DbUsername" }
      MasterUserPassword:
        Fn::Sub: ${DbPasswordDecrypt.Plaintext}
      DBSubnetGroupName: { "Ref": "ApplicationDatabaseSubnetGroup" }
      VPCSecurityGroups:
        - { "Ref" : "ApplicationDatabaseSecurityGroup" }
      MultiAZ: "False"
      AvailabilityZone:
        Fn::Sub: "${AWS::Region}a"
      Tags:
        - Key: "Name"
          Value: "intake-api-demo-ApplicationDatabase"
  ApplicationDatabaseSubnetGroup:
    Type: "AWS::RDS::DBSubnetGroup"
    Properties:
      DBSubnetGroupDescription: "intake-api-demo-ApplicationDatabase-db-subnet-group"
      SubnetIds: 
        - Fn::ImportValue: DefaultHighSubnetA
        - Fn::ImportValue: DefaultHighSubnetB
      Tags:
        - Key: "Name"
          Value: "intake-api-demo-ApplicationDatabase-db-subnet-group"
  ApplicationDatabaseSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
      GroupDescription: "intake-api-demo-ApplicationDatabase-sg"
      VpcId:
        Fn::ImportValue: DefaultVpcId
      SecurityGroupEgress:
        - IpProtocol: "icmp"
          FromPort: -1
          ToPort: -1
          CidrIp: "192.0.2.0/24"
      Tags:
        - Key: "Name"
          Value: "intake-api-demo-ApplicationDatabase-sg"
  ApplicationAutoscalingToApplicationDatabaseIngressTcp5432:
    Type: "AWS::EC2::SecurityGroupIngress"
    Properties:
      IpProtocol: "tcp"
      FromPort: "5432"
      ToPort: "5432"
      GroupId: { "Ref": "ApplicationDatabaseSecurityGroup" }
      SourceSecurityGroupId: { "Ref": "ApplicationAutoscalingSecurityGroup" }
  ApplicationAutoscalingToApplicationDatabaseEgressTcp5432:
    Type: "AWS::EC2::SecurityGroupEgress"
    Properties:
      IpProtocol: "tcp"
      FromPort: "5432"
      ToPort: "5432"
      GroupId: { "Ref": "ApplicationAutoscalingSecurityGroup" }
      DestinationSecurityGroupId: { "Ref": "ApplicationDatabaseSecurityGroup" }
  ApplicationCluster:
    Type: "AWS::ECS::Cluster"
  ApplicationTaskDefinition:
    Type: "AWS::ECS::TaskDefinition"
    Properties:
      NetworkMode: host
      Volumes:
        - Name: webroot
          Host: {}
      ContainerDefinitions:
      - Name: intake-api
        Image:
          Fn::Sub: ${ApplicationDockerImage}:${ApplicationDockerImageTag}
        MemoryReservation: 500
        LogConfiguration:
          LogDriver: awslogs
          Options:
            awslogs-group: 
              Fn::Sub: ${AWS::StackName}/ecs/IntakeApiService/intake-api
            awslogs-region: { "Ref": "AWS::Region" }
            awslogs-stream-prefix: docker
        Environment:
          - Name: RAILS_ENV
            Value: production
          - Name: PG_HOST
            Value:
              Fn::Sub: ${ApplicationDatabase.Endpoint.Address}
          - Name: DATABASE_NAME
            Value: { "Ref": "DbName" }
          - Name: PG_USER
            Value: { "Ref": "DbUsername" }
          - Name: KMS_PG_PASSWORD
            Value: { "Ref": "DbPassword" }
          - Name: KMS_SECRET_KEY_BASE
            Value: { "Ref": "SecretKeyBaseCipher" }
          - Name: ELASTICSEARCH_URL
            Value:
              Fn::Join: ["", [
                "http://",
                "Fn::Sub": "${Environment}-intake-api-es.",
                "Fn::ImportValue": "DefaultVpcDomain", ":",
                "Ref": "ElasticsearchPort"
              ] ]
          - Name: AWS_DEFAULT_REGION
            Value: { "Ref": "AWS::Region" }
          - Name: http_proxy
            Value:
              Fn::ImportValue: "DefaultProxyURL"
          - Name: https_proxy
            Value:
              Fn::ImportValue: "DefaultProxyURL"
          - Name: no_proxy
            Value:
              Fn::Join: ["", [
                "169.254.169.254,",
                "Fn::Sub": "${Environment}-intake-api-es.",
                "Fn::ImportValue": "DefaultVpcDomain"
              ] ]
          - Name: CLEAR_PROXY
            Value: "true"
        MountPoints:
          - SourceVolume: webroot
            ContainerPath: /tmp
        Command:
          - bundle
          - exec
          - puma
          - -v
          - -e
          - production
          - -b
          - unix:///tmp/app.sock
          - -C
          - config/puma.rb
      - Name: nginx
        Image:
          Fn::Sub: ${NginxDockerImage}:${NginxDockerImageTag}
        MemoryReservation: 200
        LogConfiguration:
          LogDriver: awslogs
          Options:
            awslogs-group: 
              Fn::Sub: ${AWS::StackName}/ecs/IntakeApiService/nginx
            awslogs-region: { "Ref": "AWS::Region" }
        PortMappings:
        - ContainerPort: { "Ref": "ApplicationPort" }
          Protocol: tcp
        Environment:
          - Name: HTTP_PORT 
            Value: { "Ref": "ApplicationPort" }
          - Name: WEB_ROOT
            Value: /intake_api_prototype/public
          - Name: UPSTREAM_URL
            Value: unix:///tmp/app.sock
          - Name: HEALTHCHECK
            Value: 
              Fn::Sub: curl -fs localhost:${ApplicationPort}/api/v1/screenings
        MountPoints:
          - SourceVolume: webroot
            ContainerPath: /tmp
        VolumesFrom:
          - SourceContainer: intake-api
            ReadOnly: "true"
  AdhocTaskDefinition:
    Type: "AWS::ECS::TaskDefinition"
    Properties:
      NetworkMode: host
      ContainerDefinitions:
      - Name: intake-api
        Image:
          Fn::Sub: ${ApplicationDockerImage}:${ApplicationDockerImageTag}
        MemoryReservation: 100
        LogConfiguration:
          LogDriver: awslogs
          Options:
            awslogs-group: 
              Fn::Sub: ${AWS::StackName}/ecs/IntakeApiService/adhoc
            awslogs-region: { "Ref": "AWS::Region" }
        Environment:
          - Name: RAILS_ENV
            Value: production
          - Name: PG_HOST
            Value:
              Fn::Sub: ${ApplicationDatabase.Endpoint.Address}
          - Name: DATABASE_NAME
            Value: { "Ref": "DbName" }
          - Name: PG_USER
            Value: { "Ref": "DbUsername" }
          - Name: KMS_PG_PASSWORD
            Value: { "Ref": "DbPassword" }
          - Name: ELASTICSEARCH_URL
            Value:
              Fn::Join: ["", [
                "http://",
                "Fn::Sub": "${Environment}-intake-api-es.",
                "Fn::ImportValue": "DefaultVpcDomain", ":",
                "Ref": "ElasticsearchPort"
              ] ]
          - Name: AWS_DEFAULT_REGION
            Value: { "Ref": "AWS::Region" }
          - Name: http_proxy
            Value:
              Fn::ImportValue: "DefaultProxyURL"
          - Name: https_proxy
            Value:
              Fn::ImportValue: "DefaultProxyURL"
          - Name: no_proxy
            Value:
              Fn::Join: ["", [
                "169.254.169.254,",
                "Fn::Sub": "${Environment}-intake-api-es.",
                "Fn::ImportValue": "DefaultVpcDomain"
              ] ]
          - Name: CLEAR_PROXY
            Value: "true"
  IntakeApiService:
    Type: "AWS::ECS::Service"
    DependsOn:
      - ApplicationLoadBalancer
      - ApplicationAutoscaling
      - IntakeApiServiceLogGroup
      - IntakeApiNginxLogGroup
      - DbCreateTask
      - DbMigrateTask
      - SearchMigrateTask
      - SearchReindexTask
    Properties:
      Cluster: { "Ref": "ApplicationCluster" }
      TaskDefinition: { "Ref": "ApplicationTaskDefinition" }
      DesiredCount: { "Ref": "ApplicationAutoscalingDesiredCount"}
      DeploymentConfiguration:
          MinimumHealthyPercent: 50
          MaximumPercent: 200
      LoadBalancers:
        - ContainerName: nginx
          ContainerPort: { "Ref": "ApplicationPort" }
          TargetGroupArn: { "Ref": "IntakeApiServiceTargetGroup" }
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
  ElasticsearchCluster:
    Type: "AWS::ECS::Cluster"
  ElasticsearchTaskDefinition:
    Type: "AWS::ECS::TaskDefinition"
    Properties:
      NetworkMode: host
      Volumes:
        - Name: esdata
          Host: 
            SourcePath: /ecs/esdata
      ContainerDefinitions:
      - Name: elasticsearch
        Image:
          Fn::Sub: ${ElasticsearchDockerImage}:${ElasticsearchDockerImageTag}
        MemoryReservation: 500
        LogConfiguration:
          LogDriver: awslogs
          Options:
            awslogs-group: 
              Fn::Sub: ${AWS::StackName}/ecs/ElasticsearchService/elasticsearch
            awslogs-region: { "Ref": "AWS::Region" }
        PortMappings:
        - ContainerPort: { "Ref": "ElasticsearchPort" }
          Protocol: tcp
        MountPoints:
          - SourceVolume: esdata
            ContainerPath: /var/lib/elasticsearch/data
  ElasticsearchService:
    Type: "AWS::ECS::Service"
    DependsOn:
      - ApplicationLoadBalancer
      - ElasticsearchAutoscaling
      - ElasticsearchServiceLogGroup
    Properties:
      Cluster: { "Ref": "ElasticsearchCluster" }
      TaskDefinition: { "Ref": "ElasticsearchTaskDefinition" }
      DesiredCount: { "Ref": "ElasticsearchAutoscalingDesiredCount"}
      DeploymentConfiguration:
          MinimumHealthyPercent: 0
          MaximumPercent: 200
      LoadBalancers:
        - ContainerName: elasticsearch
          ContainerPort: { "Ref": "ElasticsearchPort" }
          TargetGroupArn: { "Ref": "ElasticsearchServiceTargetGroup" }
      Role: { "Ref": "EcsServiceRole" }
  DbCreateTask:
    Type: "Custom::ECSTask"
    DependsOn:
      - ApplicationAutoscaling
      - ApplicationDatabase
    Properties:
      ServiceToken:
        Fn::Sub: ${EcsTaskRunner.Arn}
      Cluster: { "Ref": "ApplicationCluster" }
      TaskDefinition: { "Ref": "AdhocTaskDefinition" }
      Count: 1
      Overrides:
        containerOverrides:
          - name: intake-api
            command:
              - bundle
              - exec
              - rake
              - db:create
  DbMigrateTask:
    Type: "Custom::ECSTask"
    DependsOn:
      - DbCreateTask
    Properties:
      ServiceToken:
        Fn::Sub: ${EcsTaskRunner.Arn}
      Cluster: { "Ref": "ApplicationCluster" }
      TaskDefinition: { "Ref": "AdhocTaskDefinition" }
      Count: 1
      Overrides:
        containerOverrides:
          - name: intake-api
            command:
              - bundle
              - exec
              - rake
              - db:migrate
  SearchMigrateTask:
    Type: "Custom::ECSTask"
    DependsOn:
      - DbMigrateTask
      - ElasticsearchService
    Properties:
      ServiceToken:
        Fn::Sub: ${EcsTaskRunner.Arn}
      Cluster: { "Ref": "ApplicationCluster" }
      TaskDefinition: { "Ref": "AdhocTaskDefinition" }
      Count: 1
      Overrides:
        containerOverrides:
          - name: intake-api
            command:
              - bundle
              - exec
              - rake
              - search:migrate
  SearchReindexTask:
    Type: "Custom::ECSTask"
    DependsOn:
      - SearchMigrateTask
    Properties:
      ServiceToken:
        Fn::Sub: ${EcsTaskRunner.Arn}
      Cluster: { "Ref": "ApplicationCluster" }
      TaskDefinition: { "Ref": "AdhocTaskDefinition" }
      Count: 1
      Overrides:
        containerOverrides:
          - name: intake-api
            command:
              - bundle
              - exec
              - rake
              - search:reindex
  DmesgLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName: 
        Fn::Sub: ${AWS::StackName}/ec2/ApplicationAutoscaling/var/log/dmesg
      RetentionInDays: { "Ref": "LogRetention" }
  DockerLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ec2/ApplicationAutoscaling/var/log/docker
      RetentionInDays: { "Ref": "LogRetention" }
  EcsAgentLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName: 
        Fn::Sub: ${AWS::StackName}/ec2/ApplicationAutoscaling/var/log/ecs/ecs-agent
      RetentionInDays: { "Ref": "LogRetention" }
  EcsInitLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ec2/ApplicationAutoscaling/var/log/ecs/ecs-init
      RetentionInDays: { "Ref": "LogRetention" }
  MessagesLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ec2/ApplicationAutoscaling/var/log/messages
      RetentionInDays: { "Ref": "LogRetention" }
  IntakeApiServiceLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ecs/IntakeApiService/intake-api
      RetentionInDays: { "Ref": "LogRetention" }
  IntakeApiNginxLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ecs/IntakeApiService/nginx
      RetentionInDays: { "Ref": "LogRetention" }
  IntakeApiAdhocLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ecs/IntakeApiService/adhoc
      RetentionInDays: { "Ref": "LogRetention" }
  ElasticsearchDmesgLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName: 
        Fn::Sub: ${AWS::StackName}/ec2/ElasticsearchAutoscaling/var/log/dmesg
      RetentionInDays: { "Ref": "LogRetention" }
  ElasticsearchDockerLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ec2/ElasticsearchAutoscaling/var/log/docker
      RetentionInDays: { "Ref": "LogRetention" }
  ElasticsearchEcsAgentLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName: 
        Fn::Sub: ${AWS::StackName}/ec2/ElasticsearchAutoscaling/var/log/ecs/ecs-agent
      RetentionInDays: { "Ref": "LogRetention" }
  ElasticsearchEcsInitLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ec2/ElasticsearchAutoscaling/var/log/ecs/ecs-init
      RetentionInDays: { "Ref": "LogRetention" }
  ElasticsearchMessagesLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ec2/ElasticsearchAutoscaling/var/log/messages
      RetentionInDays: { "Ref": "LogRetention" }
  ElasticsearchServiceLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: ${AWS::StackName}/ecs/ElasticsearchService/elasticsearch
      RetentionInDays: { "Ref": "LogRetention" }
  DbPasswordDecrypt:
    Type: "Custom::KMSDecrypt"
    Properties:
      ServiceToken: 
        Fn::Sub: ${KMSDecrypter.Arn}
      Ciphertext: { "Ref": "DbPassword" }
  KMSDecrypterLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: /aws/lambda/${AWS::StackName}-cfnKmsDecrypt
      RetentionInDays: { "Ref": "LogRetention" }
  KMSDecrypter:
    Type: "AWS::Lambda::Function"
    DependsOn:
      - "KMSDecrypterLogGroup"
    Properties:
      Description: 
        Fn::Sub: "${AWS::StackName} KMS Decrypter"
      Handler: "cfn_kms_decrypt.handler"
      MemorySize: 128
      Runtime: "python2.7"
      Timeout: 300
      Role: 
        Fn::Sub: ${KMSDecrypterRole.Arn}
      FunctionName: 
        Fn::Sub: "${AWS::StackName}-cfnKmsDecrypt"
      Code:
        S3Bucket: 
          Fn::Sub: "${AWS::AccountId}-cfn-lambda"
        S3Key: "cfnKmsDecrypt.zip"
        S3ObjectVersion: { "Ref": "LambdaCfnKmsDecryptVersion" }
  KMSDecrypterRole:
    Type: "AWS::IAM::Role"
    Properties:
      Path: "/"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
        - Effect: "Allow"
          Principal: {"Service": "lambda.amazonaws.com"}
          Action: [ "sts:AssumeRole" ]
      Policies:
      - PolicyName: "KMSDecrypterPolicy"
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
          - Sid: "Decrypt"
            Effect: "Allow"
            Action:
            - "kms:Decrypt"
            - "kms:DescribeKey"
            Resource:
              Fn::ImportValue: "CfnMasterKeyArn"
          - Sid: "ManageLambdaLogs"
            Effect: "Allow"
            Action:
            - "logs:CreateLogGroup"
            - "logs:CreateLogStream"
            - "logs:PutLogEvents"
            - "logs:PutRetentionPolicy"
            - "logs:PutSubscriptionFilter"
            - "logs:DescribeLogStreams"
            - "logs:DeleteLogGroup"
            - "logs:DeleteRetentionPolicy"
            - "logs:DeleteSubscriptionFilter"
            Resource: 
              Fn::Sub: "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/lambda/${AWS::StackName}-cfnKmsDecrypt:*:*"
  EcsTaskRunnerLogGroup:
    Type: "AWS::Logs::LogGroup"
    DeletionPolicy: "Delete"
    Properties:
      LogGroupName:
        Fn::Sub: /aws/lambda/${AWS::StackName}-cfnEcsTasks
      RetentionInDays: { "Ref": "LogRetention" }
  EcsTaskRunner:
    Type: "AWS::Lambda::Function"
    DependsOn:
      - "EcsTaskRunnerLogGroup"
    Properties:
      Description: 
        Fn::Sub: "${AWS::StackName} ECS Task Runner"
      Handler: "ecs_tasks.handler"
      MemorySize: 128
      Runtime: "python2.7"
      Timeout: 300
      Role: 
        Fn::Sub: ${EcsTaskRunnerRole.Arn}
      FunctionName: 
        Fn::Sub: "${AWS::StackName}-cfnEcsTasks"
      Code:
        S3Bucket: 
          Fn::Sub: "${AWS::AccountId}-cfn-lambda"
        S3Key: "cfnEcsTasks.zip"
        S3ObjectVersion: { "Ref": "LambdaCfnEcsTasksVersion" }
  EcsTaskRunnerRole:
    Type: "AWS::IAM::Role"
    Properties:
      Path: "/"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
        - Effect: "Allow"
          Principal: {"Service": "lambda.amazonaws.com"}
          Action: [ "sts:AssumeRole" ]
      Policies:
      - PolicyName: "ECSPermissions"
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
          - Sid: "TaskDefinition"
            Effect: "Allow"
            Action:
            - "ecs:DescribeTaskDefinition"
            Resource: "*"
          - Sid: "EcsTasks"
            Effect: "Allow"
            Action:
            - "ecs:DescribeTasks"
            - "ecs:ListTasks"
            - "ecs:RunTask"
            - "ecs:StartTask"
            - "ecs:StopTask"
            - "ecs:DescribeContainerInstances"
            - "ecs:ListContainerInstances"
            Resource: "*"
            Condition:
              ArnEquals:
                ecs:cluster: 
                  Fn::Sub: "arn:aws:ecs:${AWS::Region}:${AWS::AccountId}:cluster/${ApplicationCluster}"
          - Sid: "ManageLambdaLogs"
            Effect: "Allow"
            Action:
            - "logs:CreateLogGroup"
            - "logs:CreateLogStream"
            - "logs:PutLogEvents"
            - "logs:PutRetentionPolicy"
            - "logs:PutSubscriptionFilter"
            - "logs:DescribeLogStreams"
            - "logs:DeleteLogGroup"
            - "logs:DeleteRetentionPolicy"
            - "logs:DeleteSubscriptionFilter"
            Resource: 
              Fn::Sub: "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/lambda/${AWS::StackName}-cfnEcsTasks:*:*"
```

4\. Once the playbook execution completes successfully, login to the **demo-resources** account.  You should see a new CloudFormation stack called **intake-api-demo**:

![Intake API Demo Stack](/images/intake-api-demo-stack.png)

5\. Select the **intake-api-demo** stack and click on the **Events** tab.  Note that events are shown in chronological order bottom to top.

Here we see an example of an ECS Task Definition being created, and various CloudWatch Log groups being created:
![Intake API Demo Stack Events](/images/intake-api-demo-events-1.png)

Next we see the KMS Decrypter Lambda function and an associated KMS Decrypt custom resource for decrypting the KMS encrypted database password:
![Intake API Demo Stack Events](/images/intake-api-demo-events-2.png)

Here we can see each autoscaling group instance signalling SUCCESS to create each autoscaling group, after which ECS tasks are executed sequentially to create the database schema, run any database migrations, create/update and reindex the Elasticsearch indexes, and finally the Intake API ECS service is started successfully:
![Intake API Demo Stack Events](/images/intake-api-demo-events-3.png)


6\. You can verify the Intake API is functional by querying the URL `https://demo-intake-api.demo.cloudhotspot.co/api/v1/screenings`, which should return an empty array:

```bash
$ curl -s https://demo-intake-api.demo.cloudhotspot.co/api/v1/screenings
[]
```

## Wrap Up

We established DNS delegation and created a wildcard certificate for the `demo.cloudhotspot.co` domain, and then published a number of Docker images required for the Intake API stack to the ECR repositories created earlier in this tutotial.  We published Lambda functions required to run CloudFormation custom resources in the Lambda S3 bucket we created earlier in the CloudFormation resources stack.  We then created a new environment in the Intake API playbook, learned how to encrypt secrets using the AWS KMS key created earlier in the CloudFormation resources stack, and finally successully deployed the Intake API stack.
