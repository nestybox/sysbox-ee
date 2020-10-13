<p align="center"><img alt="sysbox" src="./docs/figures/sysbox-ee-header.png" width="800x" /></p>

## Contents

-   [Introduction](#introduction)
-   [Free for Individual Developers, Paid for Enterprise](#free-for-individual-developers-paid-for-enterprise)
-   [Key Features](#key-features)
-   [Videos](#videos)
-   [Download](#download)
-   [Supported Distros](#supported-distros)
-   [Host Requirements](#host-requirements)
-   [Installing Sysbox](#installing-sysbox)
-   [Using Sysbox](#using-sysbox)
-   [Documentation](#documentation)
-   [Integration with Container Managers](#integration-with-container-managers)
-   [Troubleshooting](#troubleshooting)
-   [Filing Issues](#filing-issues)
-   [Support](#support)
-   [We want your feedback](#we-want-your-feedback)
-   [Uninstallation](#uninstallation)
-   [About Nestybox](#about-nestybox)
-   [Contact](#contact)
-   [Thank You](#thank-you)

## Introduction

**Sysbox Enterprise Edition** (Sysbox-EE) is the enterprise version of the
open-source [Sysbox container runtime](https://github.com/nestybox/sysbox),
developed by [Nestybox](https://www.nestybox.com).

Sysbox enables Docker containers to act as virtual servers capable of running
software such as Systemd, Docker, and Kubernetes in them, **seamlessly and
securely**. This implies the ability for these containers to run inner
containers (nested) while providing strong isolation from the underlying host.

Sysbox-EE uses Sysbox at its core, but adds enterprise-level features around
lifecycle, security, efficiency, scalability, and robustness. More on this
in the [features](#key-features) section.

## Features

The table below summarizes the key features of Sysbox Enterprise Edition
and compares it to the community edition (Sysbox CE).

<p align="center">
    <img alt="sysbox" src="./docs/figures/sysbox-features.png" width="1000x" />
</p>

More on the features [below](#feature-description).

If you have questions, you can reach us [here](#contact).

## Videos

We have some sample videos showing Sysbox-EE in action:

-   [Docker Sandboxing](https://asciinema.org/a/kkTmOxl8DhEZiM2fLZNFlYzbo?speed=2)

-   [Kubernetes-in-Docker](https://asciinema.org/a/V1UFSxz6JHb3rdHpGrnjefFIt?speed=1.75)

## Audience

Sysbox-EE is meant for engineers looking to use Sysbox as part of their
company's IT operations and/or looking to leverage the enterprise level
features it includes (i.e., enhancements over the Sysbox community edition).

Sysbox-EE is offered via a 30-day free trial. You can download and use
it for free during this time. Afterwards, we ask that you [contact](#contact)
Nestybox for pricing and payment information.

## System Containers

We call the containers deployed by Sysbox **system containers**, to highlight the
fact that they can run not just micro-services (as regular containers do), but
also system software such as Docker, Kubernetes, Systemd, inner containers, etc.

More on system containers [here](docs/user-guide/concepts.md#system-container).

## Features Description

Sysbox-EE includes all features of the open-source Sysbox runtime (aka core
features), plus enterprise-level features. These are described below.

### Core Features

#### Systemd-in-Docker

-   Run Systemd inside a Docker container easily, without complex container configurations.

-   Enables you to containerize apps that rely on Systemd (e.g., legacy apps).

#### Docker-in-Docker

-   Run Docker inside a container easily and without unsecure privileged containers.

-   Full isolation between the Docker inside the container and the Docker on the host.

#### Kubernetes-in-Docker

-   Deploy Kubernetes (K8s) inside containers with proper isolation (no
    privileged containers), using simple Docker images and Docker run commands
    (no need for custom Docker images with tricky entrypoints).

-   Deploy directly with `docker run` commands for full flexibility, or using a
    higher level tool (e.g., such as [kindbox](https://github.com/nestybox/kindbox)).

#### Strong container isolation

-   Root user in the system container maps to a fully unprivileged user on the host.

-   The procfs and sysfs exposed in the container are fully namespaced.

-   Programs running inside the system container (e.g., Docker, Kubernetes, etc)
    are limited to using the resources given to the system container itself.

-   Avoid the need for unsecure privileged containers.

#### Inner Container Image Preloading

-   You can create a system container image that includes inner container
    images, with a simple Dockerfile or Docker commit.

### Enterprise-level Features

#### Lifecycle

* Sysbox-EE package installer and systemd services.

#### Security

* Stronger cross-container isolation (Sysbox-EE assigns exclusive
  user-namespaces user-ID and group-ID mappings to each container).

#### Performance & Efficiency

* Sysbox EE includes optimizations for running containers in containers that are
  not present in the Sysbox community edition. This speeds up container
  deployment and significantly reduces storage overhead.

* For example, with Sysbox-EE, a 10-node Kubernetes-in-Docker cluster
  starts in ~2 minutes and consumes only 1GB of overhead. In contrast,
  the Sysbox open-source version takes 2 min 40 secs and consumes up to 10GB
  for this same cluster.

#### Scalability

* Higher efficiency means you can launch more system containers per host.

#### Robustness

* Sysbox-EE is tested and hardened for operation in production environments.

#### Feature Prioritization

* Sysbox-EE offers customers the ability to request and fast-track new features.

#### Nestybox Support

* Sysbox-EE includes official Nestybox support for bug fixes, updated, etc.

## Download

The latest release of Sysbox-EE is [here](https://github.com/nestybox/sysbox-ee/releases).

Installation instructions are below.

## Supported Distros

Sysbox-EE relies on functionality that is currently only present in Ubuntu Linux.

See the [distro compatibility doc](docs/distro-compat.md) for information on what versions
of Ubuntu kernels are supported.

We plan to add support for more distros in the future.

## Host Requirements

The Linux host on which Sysbox-EE runs must meet the following requirements:

1) It must have one of the supported Linux distros.

2) Systemd must be the system's process-manager (the default in the supported distros).

3) Docker must be [installed natively](docs/user-guide/install.md#docker-installation) (**not** with the Docker snap package).

## Installing Sysbox-EE

It's very easy:

1) Download the latest Sysbox-EE package from the [release](https://github.com/nestybox/sysbox-external/releases) page.

2) Verify that the checksum of the downloaded file fully matches the expected/published one.
   For example:

```console
$ sha256sum sysbox_0.2.0-0.ubuntu-focal_amd64.deb
736dba5645549ac0aabe11f29c6410bdbb76e717431a8a241833f20ce8b58a11  sysbox_0.2.0-0.ubuntu-focal_amd64.deb
```

3) Stop and eliminate all running Docker containers. Refer to the
[detailed](docs/user-guide/install.md) installation process for information
on how to avoid impacting existing containers.

```
$ docker stop $(docker ps -a -q) && docker container prune -f
```

If an error is returned, it simply indicates that no existing containers were
found.

4) Install the Sysbox-EE package and follow the installer instructions:

```console
$ sudo apt-get install ./sysbox_0.2.0-0.ubuntu-focal_amd64.deb -y
```

More information on the installation process can be found [here](docs/user-guide/install.md).

If you run into problems during install, see the [troubleshooting doc](docs/user-guide/troubleshoot.md).

## Using Sysbox-EE

Once Sysbox-EE is installed, you use it as follows:

```console
$ docker run --runtime=sysbox-runc --rm -it --hostname my_cont debian:latest
root@my_cont:/#
```

This launches a system container. It looks very much like a regular container,
but it's different under the hood.

In this container, you can now run system software such as Systemd, Docker,
Kubernetes, etc., seamlessly and securely, just as you would on a physical host
or virtual machine.

You can launch inner containers (and even inner privileged containers), with
strong isolation from the underlying host. No more complex docker images or
docker run commands, and no need for unsecure privileged containers.

The [Sysbox Quickstart Guide](docs/quickstart/README.md) and the [Nestybox Blog Site](https://blog.nestybox.com) have
many usage examples.

Note that if you omit the `--runtime` option, Docker will use its default `runc`
runtime to launch regular containers (rather than system containers). It's
perfectly fine to run system containers launched with Docker + Sysbox alongside
regular Docker containers; they won't conflict and can co-exist side-by-side.

## Documentation

We have several documents to help you get started and get the best out of
Sysbox-EE:

-   [Sysbox Quick Start Guide](docs/quickstart/README.md)

    -   Provides many examples for using system containers. New users
        should start here.

-   [Sysbox User Guide](docs/user-guide/README.md)

    -   Provides more detailed information on Sysbox features.

-   [Sysbox Distro Compatibility Doc](docs/distro-compat.md)

    -   Distro compatibility requirements.

-   [Issue Guidelines](docs/issue-guidelines.md)

    -   Guidelines for filing issues in the Sysbox-EE GitHub project site.

In addition, the [Nestybox blog site](https://blog.nestybox.com) has articles
on how to use system containers.

## Integration with Container Managers & Orchestrators

Though Sysbox is OCI-based (and thus compatible with OCI container managers),
it's currently only tested with Docker / containerd.

In particular, we don't yet support using Kubernetes to deploy system containers
with Sysbox (though we [plan to](#roadmap)).

## Troubleshooting

Refer to the [Troubleshooting document](docs/user-guide/troubleshoot.md)
and to the [issues](https://github.com/nestybox/sysbox-external/issues) in
the GitHub site.

Do [contact us](#contact) if you need any help.

## Filing Issues

We apologize for any problems in the product or documentation, and we appreciate
users filing issues that help us improve Sysbox-EE.

To file issues with Sysbox-EE (e.g., bugs, feature requests, documentation changes, etc.),
please refer to the [issue guidelines](docs/issue-guidelines.md) document.

## Support

Reach us at our [slack channel][slack] or at `contact@nestybox.com` for any questions.
See our [contact info](#contact) below for more options.

## We want your feedback

We love feedback, as it helps us improve Sysbox and set its future
direction.

We would much appreciate if you would take a couple of minutes to
answer the following survey:

<https://www.surveymonkey.com/r/SH8HMGY>

## Uninstallation

Prior to uninstalling Sysbox, make sure all system containers are removed.
There is a simple shell script to do this [here](scr/rm_all_syscont).

1) Uninstall Sysbox binaries plus all the associated configuration and Systemd
files:

```console
$ sudo apt-get purge sysbox -y
```

2) Remove the `sysbox` user from the system:

```console
$ sudo userdel sysbox
```

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

We thank you **very much** for using Sysbox. We hope you find it useful.

Your trust in us is very much appreciated.

\-- _The Nestybox Team_

[slack]: https://join.slack.com/t/nestybox-support/shared_invite/enQtOTA0NDQwMTkzMjg2LTAxNGJjYTU2ZmJkYTZjNDMwNmM4Y2YxNzZiZGJlZDM4OTc1NGUzZDFiNTM4NzM1ZTA2NDE3NzQ1ODg1YzhmNDQ
