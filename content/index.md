---
date: 2016-03-08T21:07:13+01:00
title: AWS CloudFormation Tutorial
type: index
weight: 0
---

## Introduction

This tutorial provides a walkthrough of the AWS CloudFormation deployment framework that has been established for Casecommons.

You will create a completely fresh account topology, implementing two AWS accounts:

- **Users Account** - a central users account where all user accounts and credentials are located.  Users must authenticate to this account and assume a role in a given resource account to manage resources in the target resource account.  In this tutorial the users account is called **demo-users** and has an account ID of **094411466117**.

- **Resource Account** - account where AWS resources for one or more applications will be created.  The deployment framework can support multiple resource accounts based upon the needs of your organization and product teams.  In this tutorial the resources account is called **demo-resources** and has an account ID of **160775127577**.

In this tutorial you will be configuring a number of AWS resources:

- [Security Resources]({{< relref "security-resources/index.md" >}})
- [CloudFormation Resources]({{< relref "cloudformation-resources/index.md" >}})
- [Network Resources]({{< relref "network-resources/index.md" >}})
- [EC2 Container Registry (ECR) Resources]({{< relref "ecr-resources/index.md" >}})
- [Web Proxy]({{< relref "web-proxy/index.md" >}})
- [Intake API Application]({{< relref "intake-api/index.md" >}})
- [Intake Accelerator Application]({{< relref "intake-accelerator/index.md" >}})

Each of the above consists of the following:

- Ansible playbook
- CloudFormation template that defines all AWS resources
- One or more environments, each environment ultimately implemented as a CloudFormation stack
- Git repository

## Prerequisites

This tutorial assumes you are starting from two empty AWS accounts and have access to the root account for each.

You will also need to ensure your [local machine]({{< relref "local-setup/index.md" >}}) is setup correctly.

## Cloning the Starter Repository

Although each of the playbooks described in the [Introduction]({{< relref "#introduction" >}}) are already defined as existing Git repositories, it is useful to understand how to create a playbook from scratch that implements the AWS deployment framework and methodology described in this tutorial.

A starter repository is published at https://github.com/casecommons/aws-starter.git, which you can use to clone a new repository, and follow the instructions in the [README](https://github.com/casecommons/aws-starter/README.md) to setup your playbook correctly.

## Demo Playbooks

Examples of the playbooks that are created in this tutorial are located in the [playbooks](https://github.com/casecommons/aws-docs/tree/master/playbooks) folder of the repository that hosts this documentation.