<p align="center"><img alt="sysbox" src="./docs/figures/sysbox-ee-header.png" width="1000" /></p>

***

**Docker advances container isolation and workloads with acquisition of Nestybox**:

Hi everyone, this is Cesar & Rodny, co-founders of [Nestybox](https://www.nestybox.com).

We are humbled and excited to announce that Nestybox is now officially part of
Docker, Inc! Docker is an excellent home for Sysbox, and this will accelerate
innovation of Sysbox to advance container isolation and workloads.

Please see this [blog](https://www.docker.com/blog/docker-acquires-nestybox-advancing-container-isolation-workloads/) and
this [Q&A](https://www.nestybox.com/docker-nestybox-qa) for more info. Thanks!
***

## Contents

*   [Introduction](#introduction)
*   [Features & Pricing](#features--pricing)
*   [Supported Distros](#supported-distros)
*   [Host Requirements](#host-requirements)
*   [Installation](#installation)
*   [Using Sysbox-EE](#using-sysbox-ee)
*   [Documentation](#documentation)
*   [Filing Issues](#filing-issues)
*   [Support](#support)
*   [About Nestybox](#about-nestybox)
*   [Contact](#contact)
*   [Thank You](#thank-you)

## Introduction

**Sysbox Enterprise Edition** (Sysbox-EE) is the enterprise version of the
open-source [Sysbox container runtime](https://github.com/nestybox/sysbox),
developed by [Nestybox](https://www.nestybox.com).

Sysbox-EE uses Sysbox at its core, but adds enterprise-level features such as:

*   Improved container isolation / security

*   Running more types of system-level workloads inside containers

*   Scalability (running more containers per host)

*   Significant performance and efficiency optimizations (for faster container deployment with reduced disk utilization)

*   Lifecycle (higher release cadence, critical bug fixes ASAP)

*   Nestybox professional support with a guaranteed SLA (rather than best effort on Sysbox)

*   Feature prioritization (Sysbox-EE feature requests are prioritized)

For these reasons, **\*\*we recommend that enterprises that wish to use Sysbox in
their IT infrastructure use Sysbox-EE\*\***.

Sysbox-EE is a drop-in replacement for Sysbox. It installs and it's used in the
exact same way, but includes the enterprise level features described above. On a
given host however, either Sysbox or Sysbox-EE must be installed, never both.

See the [next section](#features--pricing)for a comparison between Sysbox-EE and
Sysbox (aka Sysbox Community Edition or Sysbox-CE).

## Features & Pricing

Sysbox-EE is offered via a **30-day free trial** and a paid subscription after that.

Features and pricing info are shown below.

<p align="center">
    <img alt="sysbox" src="./docs/figures/sysbox-features.png" width="1000" />
</p>

(\*) For pricing purposes, a "host" is a computer (bare-metal or
virtual-machine) with up to 16 CPU cores (32 hyper threads). Per-core pricing at
**$5 per-core per-month** is also available for hosts with < 8 cores. Licensing
is per-year. Volume discounts available for 50+ per-host licenses or 350+
per-core licenses.

You can download Sysbox-EE for free and use it during the free trial
period. Afterwards, we ask that you contact Nestybox for [pricing and payment information](https://www.nestybox.com/pricing).

If you have questions, you can reach us [here](#contact).

## Supported Distros

Sysbox-EE relies on functionality available only in relatively recent Linux kernel
releases.

See the [distro compatibility doc](docs/distro-compat.md) for information about
the supported Linux distributions and the required kernel releases.

We plan to add support for more distros in the near future.

## Host Requirements

The Sysbox-EE host must meet the following requirements:

*   It must be running one of the [supported Linux distros](docs/distro-compat.md).

*   We recommend a minimum of 4 CPUs (e.g., 2 cores with 2 hyperthreads) and 4GB
    of RAM. Though this is not a hard requirement, smaller configurations may
    slow down Sysbox-EE.

## Installation

Sysbox-EE is a drop-in replacement for Sysbox, meaning that it's installed and
used in the same way.

For this reason, the documents in the [Sysbox repo](https://github.com/nestybox/sysbox/tree/master/docs)
apply equally to both Sysbox and Sysbox-EE.

Here are the links to the docs showing how to install Sysbox-EE:

*   [Installation on Docker hosts](https://github.com/nestybox/sysbox/blob/master/docs/user-guide/install-package.md#installing-sysbox-enterprise-edition-sysbox-ee)

*   [Installation on Kubernetes Clusters](https://github.com/nestybox/sysbox/blob/master/docs/user-guide/install-k8s.md#installation-of-sysbox-enterprise-edition-sysbox-ee)

## Using Sysbox-EE

Once Sysbox-EE is installed, you create a container using your container manager
or orchestrator (e.g., Docker or Kubernetes) and an image of your choice.

Docker command example:

```console
$ docker run --runtime=sysbox-runc --rm -it --hostname my_cont registry.nestybox.com/nestybox/ubuntu-bionic-systemd-docker
root@my_cont:/#
```

Kubernetes pod spec example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubu-bio-systemd-docker
  annotations:
    io.kubernetes.cri-o.userns-mode: "auto:size=65536"
spec:
  runtimeClassName: sysbox-runc
  containers:
  - name: ubu-bio-systemd-docker
    image: registry.nestybox.com/nestybox/ubuntu-bionic-systemd-docker
    command: ["/sbin/init"]
  restartPolicy: Never
```

You can choose whatever container image you want, Sysbox-EE places no requirements
on the image.

Refer to the [Documentation](#documentation) section below for further examples
on how to use Sysbox-EE.

## Documentation

The following documents in the [Sysbox repo](https://github.com/nestybox/sysbox/tree/master/docs)
show how to use Docker and Kubernetes to deploy containers with Sysbox.

These docs apply equally to both Sysbox and Sysbox-EE.

Features that are specific to Sysbox-EE are tagged with **"Sysbox-EE Feature
Highlight"** in the docs.

*   [Sysbox Quick Start Guide](https://github.com/nestybox/sysbox/blob/master/docs/quickstart/README.md)

    *   Provides many examples for using Sysbox to deploy enhanced
        containers. New users should start here.

*   [Sysbox User Guide](https://github.com/nestybox/sysbox/blob/master/docs/user-guide/README.md)

    *   Provides more detailed information on Sysbox features and troubleshooting.

In addition, the [Nestybox blog site](https://blog.nestybox.com) has articles
on how to use Sysbox to deploy containers.

## Filing Issues

We apologize for any problems in the product or documentation, and we appreciate
users filing issues that help us improve Sysbox-EE.

To file issues with Sysbox-EE (e.g., bugs, feature requests, documentation changes, etc.),
please refer to the [issue guidelines](docs/issue-guidelines.md) document.

## Security

If you find bugs or issues that may expose a Sysbox-EE vulnerability, please report
these by sending an email to security@nestybox.com. Please do not open security
issues in this repo. Thanks!

In addition, a few vulnerabilities have recently been found in the Linux kernel
that in some cases reduce or negate the enhanced isolation provided by Sysbox
containers. Fortunately they are all fixed in recent Linux kernels. See the
Sysbox User Guide's [Vulnerabilities & CVEs chapter](https://github.com/nestybox/sysbox/tree/master/docs/user-guide/security-cve.md)
for more info, and reach out on the [Sysbox Slack channel][slack] for further questions.

## Support

Reach us at our [slack channel][slack] or at `contact@nestybox.com` for any questions.
See our [contact info](#contact) below for more options.

Sysbox Enterprise customers get a guaranteed support SLA from Nestybox, and their
issues and requests are prioritized.

## About Nestybox

[Nestybox](https://www.nestybox.com) enhances the power of Linux containers.

We are developing software that enables containers to run **any type of
workload** (not just micro-services), and do so easily and securely.

Our mission is to provide users with a fast, efficient, easy-to-use, and secure
alternative to virtual machines for deploying virtual hosts on Linux.

## Contact

We are happy to help. You can reach us at:

Email: `contact@nestybox.com`

Slack: [Nestybox Slack Workspace][slack]

Phone: 1-800-600-6788

We are there from Monday-Friday, 9am-5pm Pacific Time.

## Thank You

We thank you **very much** for using Sysbox-EE. We hope you find it useful.

Your trust in us is very much appreciated!

\-- *The Nestybox Team*

[slack]: https://join.slack.com/t/nestybox-support/shared_invite/enQtOTA0NDQwMTkzMjg2LTAxNGJjYTU2ZmJkYTZjNDMwNmM4Y2YxNzZiZGJlZDM4OTc1NGUzZDFiNTM4NzM1ZTA2NDE3NzQ1ODg1YzhmNDQ
