Sysbox: System Container Runtime
================================

## About Nestybox

Nestybox expands the power of Linux containers.

We are developing software that enables deployment of **system containers**
with Docker (and soon Kubernetes).

A Nestybox system container is a Linux container designed to run low-level system
software, not just applications. See this [blog article](https://blog.nestybox.com/2019/09/13/system-containers.html) for more info on system
containers and the use cases we envision for them.

Our mission is to make our system containers run as many system-level
workload types as possible in order to provide users a fast,
efficient, and easy-to-use alternative to virtual machines for
deploying virtual hosts on Linux. And for this work out-of-the-box and
securely, without complex configurations or hacks.

## About Sysbox

Sysbox is software that installs on a Linux host and integrates with Docker,
enabling Docker to create system containers.

Users do not normally interact with Sysbox directly. Instead, users
create system containers with Docker as described below.

## Features

**NOTE**: It's early days for Nestybox, so our system containers
support a reduced set of features and use-cases at this time.

Below is a list of features currently supported by Sysbox.

### Deployment

* Supports deployment of system containers with Docker.

* The system containers can run concurrently with regular Docker
  application containers, without conflict.

### System Container Software

* Supports running Docker inside the system container.

  - Cleanly & securely, with total isolation between the Docker inside
    the container and the Docker on the host. No need to use insecure
    privileged containers, or to bind-mount the host's Docker socket
    into the container.

  - The Docker inside the system container can build and run
    containers as usual.

  - This is useful for testing & CI/CD use cases.

### Security & Isolation

* Strong system container isolation

  - System containers use the Linux user namespace and exclusive
    user-ID and group-ID mappings for increased container-to-host and
    container-to-container isolation.

* Resource isolation

  - Programs inside the system container (e.g., Docker) are limited
    to using the resources given to the system container itself.

* Partially virtualized procfs

  - Processes inside the system container see a partially virtualized `/proc`.

  - This makes the system container more closely resemble a real host.

  - Prevents processes within the container from changing global
    kernel settings.

Please see our [Roadmap](#roadmap) for a list of features we are working on.

## Supported Linux Distros

Sysbox relies on functionality that is only present in very recent
Ubuntu kernels:

* Ubuntu 19.04 "Disco" (kernel >= 5.0.0-21.22)
* Ubuntu 18.04 "Bionic" (with 5.0+ kernel upgrade)

If you need to upgrade your kernel the match requirements stated
above, see [here](docs/troubleshoot.md#upgrading-ubuntu-kernel) for
suggestions on how to do this.

Alternatively it's possible to use Sysbox with slightly older Ubuntu
kernels, but doing so requires that the Docker daemon be configured
with [userns-remap](docs/usage.md#interaction-with-docker-userns-remap).

In this case you can run Sysbox on the following distros (without the
need to upgrade the kernel):

* Ubuntu 19.04 "Disco"
* Ubuntu 18.10 "Cosmic"
* Ubuntu 18.04 "Bionic"

We plan to add support for more distros in the future.

## Host Requirements

The Linux host on which Sysbox runs must meet the following requirements:

1) It must have one of the Linux distros listed in the prior section.

2) Systemd must be the system's process-manager (the default in the supported distros).

3) Docker must be installed on the host machine.

## Installation

1) Download the latest package from the [release](https://github.com/nestybox/sysbox-external/releases) page.

2) Verify that the checksum of the downloaded file fully matches the expected/published one.
   For example:

```console
$ sha256sum ~/sysbox_0.0.1-0~ubuntu-bionic_amd64.deb
2a02898dc53b4751cf413464b977f5b296d9aac3c5b477e05272bfa881d69cfc  /home/user/sysbox_0.0.1-0~ubuntu-bionic_amd64.deb
```

3) Install the Sysbox package:

```console
$ sudo dpkg -i sysbox_0.0.1-0~ubuntu-bionic_amd64.deb
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

It's perfectly fine to run system containers along side with regular
Docker application containers on the host at the same time; they won't
conflict.

Refer to the [Sysbox User's Guide](docs/usage.md) for other ways to
run system containers with Sysbox.

If you hit problems with the instructions above, see the
[Troubleshooting document](docs/troubleshoot.md).

## Software supported inside the System Container

A system container is logically a super-set of a regular Docker
application container, and thus should be able to run any application
that runs in a regular Docker container. In addition, it runs
system-level software that does not run in a regular Docker container.

For system-level software, we currently only support running Docker
inside the system container. This allows you to build and run Docker
application containers inside the system container, just as you would
on a physical host or in a VM. It's useful in CI/CD pipelines where
the need for a container to build another container arises often.

See [here](docs/usage.md#running-software-inside-the-system-container) for more info on this.

## Integration with Container Managers

Sysbox is designed to work with Docker / containerd.

We don't yet support other container managers (e.g., cri-o).

## Design

For more detailed info about Sysbox's design, refer to the
[Sysbox design document](docs/design.md).

## OCI Compatibility

Sysbox is a fork of the [OCI runc](https://github.com/opencontainers/runc). It is mostly
(but not 100%) compatible with the OCI runtime specification. See [here](docs/design.md#oci-compatibility)
for a list of incompatibilities.

We believe these incompatibilities won't negatively affect users of
Sysbox and should mostly be transparent to them.

## Production Readiness

Sysbox is still in an experimental stage. It's **not** production ready yet.

Nestybox is actively enhancing its functionality and fixing issues at this stage.

Your feedback is much appreciated!

## Troubleshooting

Refer to the [Troubleshooting document](docs/troubleshoot.md).

## Issues

We apologize for any problems in the product or documentation, and we appreciate
customers filing issues that help us improve them.

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

* Support for more Linux distros.

* Support for Docker volume plugins for use with system containers.

* Support for deploying system containers with Kubernetes.

* Support for other container managers (e.g., cri-o)

* Running Systemd inside the system container

* Running Kubernetes inside the system container

* Running window managers (e.g., X) inside the system container (for GUI apps & desktops).

## Feedback

We love feedback, as it helps us improve Sysbox and set its future
direction.

We would much appreciate if you would take a couple of minutes to
answer the following survey:

https://www.surveymonkey.com/r/SH8HMGY

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

## Thank You!

We thank you **very much** for using Sysbox. We hope you find it useful.

Your trust in us is very much appreciated.

-- *The Nestybox Team*
