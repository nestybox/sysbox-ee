Nestybox Sysboxd
================

## About Nestybox

Nestybox is re-imagining server virtualization.

We are developing software solutions that improve efficiency,
performance, portability, and security over current Linux container
and server virtualization technologies.

## About Sysboxd

Sysboxd is software that installs on a Linux host and integrates with Docker,
enabling Docker to create **system containers**. See [here](docs/system-containers.md)
for a description of what a system container is and the use cases
we envision for them.

Users do not normally interact with Sysboxd directly. Instead, users
create system containers with Docker. See [Usage](#usage) below for more info.

## Features

**NOTE**: It's early days for Nestybox, so our system containers
support a reduced set of features and use-cases at this time.

Below is a list of features currently supported by sysboxd. Please
see our [Roadmap](#roadmap) for a list of features we are working on.

### Deployment

* Supports deployment of system containers with Docker.

* System containers can run concurrently with regular Docker
  containers, without conflict.

### System Container Software

* Supports running Docker inside the system container.

  - [Cleanly & securely](#docker-in-docker), with total isolation between the inner and
    outer Docker containers.

  - This is useful for testing & CI/CD use cases.

### Security & Isolation

* Strong system container isolation

  - System containers use the Linux user namespace and exclusive
    user-ID and group-ID mappings for increased container-to-host and
    container-to-container isolation.

* Exposes a partially virtualized procfs (`/proc`) to the container.

  - This makes the container more closely resemble a real host.

  - Prevents processes within the container from changing global kernel
    settings via `/proc`.

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

The Linux host on which sysboxd runs must meet the following requirements:

1) Systemd must be running as the system's process-manager.

2) Docker must be installed on the host machine.

3) The host's kernel must be configured to allow unprivileged users
   to create namespaces. For Ubuntu:

   ```
   sudo sh -c "echo 1 > /proc/sys/kernel/unprivileged_userns_clone"
   ```

   **Note:** This instruction will be *automatically* executed by the
   Sysboxd package installer, so there is no need for the user to
   manually type it.

4) Sysboxd stores some internal state in `/var/lib/sysboxd`. This directory
   must be on one of the following filesystems:

   * ext4
   * btrfs

   The same requirement applies to the `/var/lib/docker` directory.

   This is normally the case for vanilla Ubuntu installations.

5) If the host runs Ubuntu-Bionic, you'll need to update the Linux kernel to
   5.X+ (unless you enable docker [userns-remap](docs/usage.md#interaction-with-docker-userns-remap)).

   Note that you must use the Ubuntu 5.X+ kernel, **not** the Linux
   upstream kernel (because Ubuntu carries patches that are not
   present in the upstream kernel). The easiest way to do this is to
   use Ubuntu's [LTS-enablement](https://wiki.ubuntu.com/Kernel/LTSEnablementStack) package:

   ```
   $ sudo apt-get update && sudo apt install --install-recommends linux-generic-hwe-18.04 -y
   ```

## Installation

1) Download the latest package from the [release](https://github.com/nestybox/sysboxd-external/releases) page.

2) Verify that the checksum of the downloaded file fully matches the expected/published one:

```bash
$ sha256sum ~/sysboxd_0.0.1-0~ubuntu-bionic_amd64.deb
2a02898dc53b4751cf413464b977f5b296d9aac3c5b477e05272bfa881d69cfc  /home/user/sysboxd_0.0.1-0~ubuntu-bionic_amd64.deb
```

3) Install the Sysboxd package:

```bash
$ sudo dpkg -i sysboxd_0.0.1-0~ubuntu-bionic_amd64.deb
```

In case you hit an error with missing dependencies, fix this with:

```bash
$ sudo apt-get install -f -y
```

This will install the missing dependencies and automatically re-launch
the Sysboxd installation process.


4) Verify that Sysboxd's systemd units have been properly installed, and
   associated daemons are properly running:

```
$ systemctl list-units -t service --all | grep sysbox
sysbox-fs.service                   loaded    active   running sysbox-fs component
sysbox-mgr.service                  loaded    active   running sysbox-mgr component
sysboxd.service                     loaded    active   exited  Sysboxd General Service
```

The sysboxd.service is ephemeral (it exits once it launches the other sysboxd services).

If you are curious on what the other Sysboxd services are, refer to the [Sysboxd design document](docs/design.md).

If you hit problems during installation, see the [Troubleshooting document](docs/troubleshoot.md).

## Usage

To launch a system container with Docker, point Docker to the Sysboxd container
runtime, using the `--runtime=sysbox-runc` option:

```bash
$ docker run --runtime=sysbox-runc --rm -it --hostname my_cont debian:latest
root@my_cont:/#
```

If you omit the `--runtime` option, Docker will use its default runc
runtime to launch regular application containers (rather than system
containers).

It's perfectly fine to run system containers along side with regular
Docker application containers on the host at the same time; they won't
conflict.

Refer to the [Sysboxd User's Guide](docs/usage.md) for other ways to
run system containers with Sysboxd.

## Software supported inside the System Container

A system container is logically a super-set of a regular Docker
application container, and thus should be able to run any application
that runs in a regular Docker container. In addition, it runs
system-level software that does not run in a regular Docker container.

For system-level software, we currently only support running Docker
inside the system container. See [here](docs/usage.md#running-software-inside-the-system-container)
for more info on this.

## Integration with Container Managers

Sysboxd is designed to work with Docker / containerd.

We don't yet support other container managers (e.g., cri-o).

## Design

For more detailed info about Sysboxd's design, refer to the
[Sysboxd design document](docs/design.md).

## OCI Compatibility

Sysboxd is a fork of the [OCI runc](https://github.com/opencontainers/runc). It is mostly
(but not 100%) compatible with the OCI runtime specification. See [here](docs/design.md#oci-compatibility)
for a list of incompatibilities.

We believe these incompatibilities won't negatively affect users of
Sysboxd and should mostly be transparent to them.

## Production Readiness

Sysboxd is still in Beta. It's stable, but not production ready yet.

## Troubleshooting

Refer to the [Troubleshooting document](docs/troubleshoot.md).

## Issues

We apologize for any problems in the product or documentation, and we appreciate
customers filing issues that help us improve them.

To file issues with Sysboxd (e.g., bugs, feature requests, documentation changes, etc.),
please refer to the [issue guidelines](docs/issue-guidelines.md) document.

## Roadmap

The following is a list of features in the Sysboxd roadmap.

We list these here so that our users can get a better idea of where we
are going and can give us feedback on which of these they like best
(or least).

Nestybox reserves the right to change these based on business
priorities.

Here is the list:

* Support for more Linux distros.

* Support for Docker volume plugins for use with system containers.

* Support for other container managers (e.g., cri-o)

* Running Kubernetes inside the system container

* Running Systemd inside the system container

* Running window managers (e.g., X) inside the system container (for GUI apps & desktops).

* More virtualization of non-namespaced resources in procfs (`/proc/`).

## Feedback

We love feedback, as it helps us improve Sysboxd and set its future
direction.

We would much appreciate if you would take a couple of minutes to
answer the following survey:

https://www.surveymonkey.com/r/SH8HMGY

## Uninstallation

Remove all installed binaries (including the 'nbox-shiftfs' kernel submodule):

```bash
$ sudo dpkg --remove sysboxd
```

Alternatively, remove the above items plus all the associated
configuration/systemd files (recommended):

```bash
$ sudo dpkg --purge sysboxd
```

## Thank You!

We thank you **very much** for using Sysboxd. We hope you find it useful.

Your trust in us is very much appreciated.

-- *The Nestybox Team*
