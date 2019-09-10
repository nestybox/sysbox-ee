Sysbox: System Container Runtime
================================

## About Nestybox

Nestybox expands the power of Linux containers.

We are developing software that enables deployment of **system containers**
with Docker (and soon Kubernetes).

A system container is a Linux container designed to run low-level system
software, not just applications. See [here](docs/system-containers.md) for more info on system
containers and the use cases we envision for them.

Our mission is to make system containers run as many system-level
workload types as possible in order to provide users a fast,
efficient, and easy-to-use alternative to virtual machines for
deploying virtual hosts on Linux. And for this work out-of-the-box and
securely, without complex configurations or hacks.

## About Sysbox

Sysbox is software that installs on a Linux host and integrates with Docker,
enabling Docker to create [system containers](docs/system-containers.md).

Users do not normally interact with Sysbox directly. Instead, users
create system containers with Docker. See [Usage](#usage) below for more info.

## Features

**NOTE**: It's early days for Nestybox, so our system containers
support a reduced set of features and use-cases at this time.

Below is a list of features currently supported by Sysbox. Please
see our [Roadmap](#roadmap) for a list of features we are working on.

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

## Supported Linux Distros

* Ubuntu 19.04 "Disco"
* Ubuntu 18.04 "Bionic" (kernel upgrade required; see [Host Requirements](#host-requirements) below)

The supported distros increase when Docker is configured with
[userns-remap](docs/usage.md#interaction-with-docker-userns-remap) enabled. In this case, the supported distros are:

* Ubuntu 19.04 "Disco"
* Ubuntu 18.10 "Cosmic"
* Ubuntu 18.04 "Bionic"

We plan to add support for more distros in the future.

## Host Requirements

The Linux host on which Sysbox runs must meet the following requirements:

1) Systemd must be running as the system's process-manager.

2) Docker must be installed on the host machine.

3) If the host runs Ubuntu-Bionic, you'll need to update the Linux kernel to
   5.X+ (unless you enable docker [userns-remap](docs/usage.md#interaction-with-docker-userns-remap)).

   Note that you must use the Ubuntu 5.X+ kernel, **not** the Linux
   upstream kernel (because Ubuntu carries patches that are not
   present in the upstream kernel). The easiest way to do this is to
   use Ubuntu's [LTS-enablement](https://wiki.ubuntu.com/Kernel/LTSEnablementStack) package:

   ```
   $ sudo apt-get update && sudo apt install --install-recommends linux-generic-hwe-18.04 -y
   ```

## Installation

1) Download the latest package from the [release](https://github.com/nestybox/sysbox-external/releases) page.

2) Verify that the checksum of the downloaded file fully matches the expected/published one.
   For example:

```bash
$ sha256sum ~/sysbox_0.0.1-0~ubuntu-bionic_amd64.deb
2a02898dc53b4751cf413464b977f5b296d9aac3c5b477e05272bfa881d69cfc  /home/user/sysbox_0.0.1-0~ubuntu-bionic_amd64.deb
```

3) Install the Sysbox package:

```bash
$ sudo dpkg -i sysbox_0.0.1-0~ubuntu-bionic_amd64.deb
```

In case you hit an error with missing dependencies, fix this with:

```bash
$ sudo apt-get install -f -y
```

This will install the missing dependencies and automatically re-launch
the Sysbox installation process.


4) Verify that Sysbox's systemd units have been properly installed, and
   associated daemons are properly running:

```
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

```bash
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

Sysbox is still in experimental stage. It's **not** production ready yet.

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

* Running Kubernetes inside the system container

* Running Systemd inside the system container

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

```bash
$ sudo dpkg --remove sysbox
```

Alternatively, remove the above items plus all the associated
configuration and systemd files (recommended):

```bash
$ sudo dpkg --purge sysbox
```

2) Unload the `nbox_shiftfs` module:

```bash
$ sudo rmmod nbox_shiftfs
```

3) Finally remove the `sysbox` user from the system:

```bash
$ sudo userdel sysbox
```

## Contact

Please contact us at `contact@nestybox.com` for any questions. We will
be happy to help.

## Thank You!

We thank you **very much** for using Sysbox. We hope you find it useful.

Your trust in us is very much appreciated.

-- *The Nestybox Team*
