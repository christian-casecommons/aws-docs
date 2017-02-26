---
date: 2017-02-24T22:14:44+13:00
title: Local Setup
weight: 10
---

{{< note title="Note" >}}
The instructions in this tutorial assume you are using a newly install macOS environment.  You will need to adapt the instructions accordingly if you are installing in a Linux environment.
{{< /note >}}

To run the deployment framework from your local machine, you need to install a number of software components including:

- Homebrew
- Python (Homebrew package)
- git 2.11 or higher (Homebrew package)
- jq 1.5 or higher (Homebrew package)
- Ansible 2.2 or higher (PIP package)
- AWS CLI 1.11 or higher (PIP package)
- netaddr (PIP package)
- ndg-httpsclient (PIP package)
- boto (PIP package installed using macOS system Python)

To run Docker related tasks in this tutorial you need to install the following components:

- Docker 1.12 or higher
- Docker Compose 1.9 or higher
- GNU Make
- Access to a working Docker Engine (e.g. Docker for Mac or Docker Machine)

## Installation Instructions

1.  To install Homebrew, browse to http://brew.sh and follow the installation instructions.

2.  Once Homebrew is installed, you can install Python, git and jq using Homebrew as follows:

    ```bash
    $ brew install python git jq
    ...
    ```

3.  To install Ansible, AWS CLI and other required Python packages, it is recommended to use PIP as follows:

    ```bash
    $ pip install ansible awscli netaddr ndg-httpsclient
    ...
    ...
    ```

4.  To install the `boto` package, you need to install both `pip` and `boto` using the macOS system Python installation:

    ```bash
    $ sudo -H /usr/bin/python -m easy_install pip
    ...
    ...
    $ sudo -H /usr/bin/python -m pip install boto
    ...
    ...
    ```