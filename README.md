# Sysbox: System Container Runtime

## Contents

-   [About Nestybox](#about-nestybox)
-   [About Sysbox](#about-sysbox)
-   [Features](#features)
    -   [System Container Deployment](#system-container-deployment)
    -   [System Container Software](#system-container-software)
    -   [System Container Image Creation](#system-container-image-creation)
    -   [Security and Isolation](#security-and-isolation)
-   [Supported Linux Distros](#supported-linux-distros)
-   [Host Requirements](#host-requirements)
-   [Installation](#installation)
-   [Usage](#usage)
-   [Documentation](#documentation)
-   [Software supported inside the System Container](#software-supported-inside-the-system-container)
-   [Integration with Container Managers](#integration-with-container-managers)
-   [Production Readiness](#production-readiness)
-   [Troubleshooting](#troubleshooting)
-   [Issues](#issues)
-   [Roadmap](#roadmap)
-   [We need your feedback](#we-need-your-feedback)
-   [Uninstallation](#uninstallation)
-   [Contact](#contact)
-   [Thank You](#thank-you)

## About Nestybox

Nestybox expands the power of Linux containers.

We are developing software that enables deployment of **system containers**
with Docker (and soon Kubernetes).

A Nestybox system container is a Linux container designed to run low-level system
software, not just applications. See this [blog article](https://blog.nestybox.com/2019/09/13/system-containers.html) for more info on system
containers and some of the use cases we envision for them.

Our mission is to make our system containers run as many system-level
workload types as possible in order to provide users a fast,
efficient, and easy-to-use alternative to virtual machines for
deploying virtual hosts on Linux. And for this to work out-of-the-box
and securely, without complex configurations and without resorting
to unsecure privileged containers.

## About Sysbox

Sysbox is software that installs on a Linux host and integrates with Docker,
enabling Docker to create system containers.

Users do not normally interact with Sysbox directly. Instead, users
create system containers with Docker as described below.

## Features

**NOTE**: It's early days for Nestybox, so our system containers
support a reduced set of features and use-cases at this time.

Below is a list of features currently supported by Sysbox.

### System Container Deployment

-   Supports deployment of system containers with Docker.

-   The system containers can run concurrently with regular Docker
    application containers, without conflict.

### System Container Software

-   Supports running Docker inside the system container.

    -   Cleanly & securely, with total isolation between the Docker inside
        the container and the Docker on the host. No need to use unsecure
        privileged containers or to bind-mount the host's Docker socket
        into the container.

    -   The Docker inside the system container can build and run
        containers as usual.

    -   This is useful for Docker sandboxing, testing and CI/CD use cases.

-   Supports running Systemd inside the system container (preliminary support).

    -   Useful for system containers that are used as virtual hosts.

    -   Run Systemd securely (without resorting to privileged Docker containers).

    -   Super easy: simply launch a system container image with Systemd as
        its entry point and Sysbox will ensure the system container is setup
        to run Systemd without problems.

### System Container Image Creation

-   Use Docker to build system container images, just like regular containers.

-   In addition, Sysbox supports using `docker build` or `docker commit` to create
    system container images with pre-packaged inner containers in them.

    -   This enables you to use the system container as a fully pre-configured
        Docker sandbox environment.

    -   When you start the system container all inner Docker container images
        are ready to run. No need to pull the inner Docker images from a
        remote repository.

### Security and Isolation

-   Enhanced system container isolation

    -   System containers use the Linux user namespace and exclusive
        user-ID and group-ID mappings for increased container-to-host and
        container-to-container isolation.

-   Resource isolation

    -   Programs inside the system container (e.g., Docker) are limited
        to using the resources given to the system container itself.

-   Partially virtualized procfs

    -   Processes inside the system container see a partially virtualized `/proc`.

    -   This makes the system container more closely resemble a physical
        host or VM.

    -   Prevents processes within the container from changing global
        kernel settings.

Please see our [Roadmap](#roadmap) for a list of features we are working on.

## Supported Linux Distros

Sysbox relies on functionality that is only present in very recent
Ubuntu kernels:

-   Ubuntu 19.04 "Disco" (kernel >= 5.0.0-21.22)
-   Ubuntu 18.04 "Bionic" (with 5.0+ kernel upgrade)

If you need to upgrade your kernel in order to match requirements
stated above, see [here](docs/troubleshoot.md#upgrading-the-ubuntu-kernel)
for suggestions on how to do this.

Alternatively it's possible to use Sysbox with slightly older Ubuntu
kernels, but doing so requires configuring Sysbox in "Docker
userns-remap isolation mode". In this case you can run Sysbox on the
following distros (without needing to upgrade the kernel):

-   Ubuntu 19.04 "Disco"
-   Ubuntu 18.10 "Cosmic"
-   Ubuntu 18.04 "Bionic"

We plan to add support for more distros in the future.

Here is a summary of the distro requirements:

| System Container Isolation Mode | Required Distro & Kernel            |
| ------------------------------- | ----------------------------------- |
| Exclusive userns-remap          | Ubuntu 19.04 Disco (>= 5.0.0-21.22) |
|                                 | Ubuntu 18.04 Bionic (5.0+ kernel)   |
| Docker userns-remap             | Ubuntu 19.04 Disco                  |
|                                 | Ubuntu 18.10 Cosmic                 |
|                                 | Ubuntu 18.04 Bionic                 |

See [here](docs/usage.md#system-container-isolation-modes) for info on
Sysbox isolation modes and how to configure them.

## Host Requirements

The Linux host on which Sysbox runs must meet the following requirements:

1) It must have one of the Linux distros listed in the prior section.

2) Systemd must be the system's process-manager (the default in the supported distros).

3) Docker must be installed.

## Installation

1) Download the latest Sysbox package from the [release](https://github.com/nestybox/sysbox-external/releases) page.

2) Verify that the checksum of the downloaded file fully matches the expected/published one.
   For example:

```console
$ sha256sum sysbox_0.1.2-0.ubuntu-disco_amd64.deb
23b99987bd0c5fb347f0231a47e4dc7c27bd082baaac942055ce6168adf6d9e2  sysbox_0.1.2-0.ubuntu-disco_amd64.deb
```

3) Install the Sysbox package:

```console
$ sudo dpkg -i sysbox_0.1.2-0.ubuntu-disco_amd64.deb
```

In case you hit an error with missing dependencies, fix this with:

```console
$ sudo apt-get install -f -y
```

This will install the missing dependencies and automatically re-launch
the Sysbox installation process.

4) Verify that Sysbox's systemd units have been properly installed, and
   associated daemons are properly running:

```console
$ systemctl list-units -t service --all | grep sysbox
sysbox-fs.service                   loaded    active   running sysbox-fs component
sysbox-mgr.service                  loaded    active   running sysbox-mgr component
sysbox.service                     loaded    active   exited  Sysbox General Service
```

Note: the sysbox.service is ephemeral (it exits once it launches the other sysbox services).

If you are curious on what the other Sysbox services are, refer to the [Sysbox design document](docs/design.md).

If you hit problems during installation, see the [Troubleshooting document](docs/troubleshoot.md).

## Usage

To launch a system container with Docker, point Docker to the Sysbox container
runtime, using the `--runtime=sysbox-runc` option:

```console
$ docker run --runtime=sysbox-runc --rm -it --hostname my_cont debian:latest
root@my_cont:/#
```

If you omit the `--runtime` option, Docker will use its default `runc`
runtime to launch regular application containers (rather than system
containers).

It's perfectly fine to run system containers launched with Docker +
Sysbox along side regular Docker application containers; they won't
conflict.

## Documentation

We have several documents to help you use and get the best out of
system containers.

-   [Sysbox Quick Start Guide](docs/quickstart.md)

    -   Provides many examples for using system containers. New users
        should start here.

-   [Sysbox User's Guide](docs/usage.md)

    -   Provides more detail information on Sysbox features.

-   [Sysbox Design Notes](docs/design.md)

    -   Provides information on Sysbox's design.

-   [Sysbox Security Guide](docs/security.md)

    -   Provides information on system container security.

-   [Troubleshooting Guide](docs/troubleshoot.md)

    -   Refer to this document if you hit problems.

-   [Issue Guidelines](docs/issue-guidelines.md)

    -   Guidelines for filing issues in the Sysbox GitHub project site.

Also, the [Nestybox blog site](https://blog.nestybox.com) has articles
on how to use system containers.

## Software supported inside the System Container

A system container is logically a super-set of a regular Docker
application container, and thus should be able to run any application
that runs in a regular Docker container. In addition, it runs
system-level software that does not run in a regular Docker container.

For system-level software, we currently support running the following
inside the system container:

-   Systemd

    -   Allows using the system container as a virtual host, much like you
        would use a VM.

-   Docker

    -   Allows you to build and run Docker application containers inside
        the system container, just as you would on a physical host or in a
        VM.

    -   Allows you to use the system container as a Docker sandbox, or in
        CI/CD pipelines where the need to deploy a container to build
        another container arises often.

See [here](docs/usage.md#running-software-inside-the-system-container) for more info on this.

## Integration with Container Managers

Sysbox is designed to work with Docker / Containerd.

We don't yet support other container managers (e.g., cri-o, etc).

## Production Readiness

Sysbox is still in alpha / experimental stage. It's **not production ready yet**.

Nestybox is actively enhancing its functionality and fixing issues at this time.

Your feedback regarding issues or improvements is much appreciated!

## Troubleshooting

Refer to the [Troubleshooting document](docs/troubleshoot.md).

## Issues

We apologize for any problems in the product or documentation, and we appreciate
customers filing issues that help us improve Sysbox.

To file issues with Sysbox (e.g., bugs, feature requests, documentation changes, etc.),
please refer to the [issue guidelines](docs/issue-guidelines.md) document.

## Roadmap

The following is a list of features in the Sysbox roadmap.

We list these here so that our users can get a better idea of where we
are going and can give us feedback on which of these they like best
(or least).

Nestybox reserves the right to change these based on business
priorities.

Here is the list:

-   Support for more Linux distros.

-   Support for deploying system containers with Kubernetes.

-   Support for other container managers (e.g., cri-o).

-   Running Kubernetes inside the system container.

-   Exposing host devices within the system container.

-   Running window managers (e.g., X) inside the system container (for GUI apps & desktops).

## We need your feedback

We love feedback, as it helps us improve Sysbox and set its future
direction.

We would much appreciate if you would take a couple of minutes to
answer the following survey:

<https://www.surveymonkey.com/r/SH8HMGY>

## Uninstallation

Prior to uninstalling Sysbox, make sure all system containers are removed.
There is a simple shell script to do this [here](scr/rm_all_syscont).

1) Uninstall Sysbox binaries:

```console
$ sudo dpkg --remove sysbox
```

Alternatively, remove the above items plus all the associated
configuration and systemd files (recommended):

```console
$ sudo dpkg --purge sysbox
```

2) Remove the `sysbox` user from the system:

```console
$ sudo userdel sysbox
```

## Contact

Please contact us at `contact@nestybox.com` for any questions. We will
be happy to help.

## Thank You

We thank you **very much** for using Sysbox. We hope you find it useful.

Your trust in us is very much appreciated.

\-- _The Nestybox Team_
