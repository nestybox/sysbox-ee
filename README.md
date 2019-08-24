Nestybox Sysboxd
================

## About Nestybox

Nestybox is re-imagining server virtualization.

We are developing software solutions that improve efficiency,
performance, portability, and security over current Linux container
and server virtualization technologies.

## About Sysboxd

Sysboxd is software that integrates with Docker and allows it to
create [system containers](docs/system-containers.md).

Sysboxd installs on a Linux host and it's normally not used directly.
Instead, users create containers using Docker by pointing it to use the
Sysboxd container runtime component (sysbox-runc). See [Usage](#usage)
below for more info.

## Features

### Deployment

* Supports deployment of system containers with Docker.

* System containers can run concurrently with regular Docker
  containers, without conflict.

### System Container Software

* Supports running Docker inside the system container.

  - Securely, with total isolation between the Docker inside the
    container and the Docker on the host (e.g,. without using Docker
    privileged containers on the host and without bind mounting the
    host's Docker socket into the container).

  - Fast: the inner Docker uses the overlayfs driver.

  - This is useful for testing & CI/CD use cases.

### Security & Isolation

* Strong system container isolation

  - System containers use the Linux user namespace and exclusive
    user-ID and group-ID mappings, for increased container-to-host and
    contanier-to-container isolation.

  - This means System containers are more secure than regular Docker
    application containers.

* Exposes a partially virtualized procfs (`/proc`) to the container.

  - This makes the container more closely resemble a real host.

  - Prevents processes within the container from changing global kernel
    settings via `/proc`.

**NOTE**: It's early days for Nestybox, so our system containers support a
reduced set of features and [use-cases](docs/system-containers.md#use-cases) at this time.
Please see our [Roadmap](#roadmap) for a list of features we are working on.

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

* Docker must be installed on the host machine.

* Systemd must be running as the system's process-manager.

* The host's kernel must be configured to allow unprivileged users
  to create namespaces. For Ubuntu:

```
sudo sh -c "echo 1 > /proc/sys/kernel/unprivileged_userns_clone"
```

  Note: This instruction will be *automatically* executed by the
  Sysboxd package installer, so there is no need for the user to
  manually type it.

* If the host runs Ubuntu-Bionic, you'll need to update the Linux kernel to
  5.X+ (unless you enable docker [userns-remap](docs/usage.md#interaction-with-docker-userns-remap)).

  Note that you must use the Ubuntu 5.X+ kernel, **not** the Linux upstream kernel (because
  Ubuntu carries patches that are not present in the upstream kernel). The easiest way to do
  this is to use Ubuntu's [LTS-enablement](https://wiki.ubuntu.com/Kernel/LTSEnablementStack)
  package:

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

If you are curious on what these Sysboxd services are, refer to the [Sysboxd design document](docs/design.md).

If you hit problems during installation, see the [Troubleshooting document](docs/troubleshoot.md).

## Usage

To launch a system container with Docker, simply use the Docker
`--runtime=sysbox-runc` flag:

```bash
$ docker run --runtime=sysbox-runc --rm -it --hostname my_cont debian:latest
root@my_cont:/#
```

If you omit the `--runtime` flag, Docker will use the default runc
runtime to launch regular application containers (rather than system containers).

It's perfectly fine to run system containers along side with regular
Docker application containers on the host at the same time; they won't
conflict.

Refer to the [Sysboxd User's Guide](docs/usage.md) for other ways to
run system containers with Sysboxd.

## Software supported inside the System Container

A system container is logically a super-set of a regular Docker
application container, and thus should be able to run any application
that runs in a regular Docker container, plus system-level software.

Nestybox's goal is to allow you run any software inside the system
container just as you would on a physical host. Ideally there
shouldn't be any difference.

### Docker-in-Docker

Nestybox system containers support running Docker inside the system
container, without using privileged containers or bind-mounting the
host's Docker socket into the container. In other words, cleanly and
securely, with total isolation between the inner and outer Docker
containers.

Furthermore, the inner Docker can use the fast overlayfs driver,
rather than the slower vfs driver.

To run Docker-in-Docker, follow the instructions in the [Sysboxd User's Guide](docs/usage.md#docker-in-docker).

## Integration with Container Managers

Sysboxd is designed to work with Docker / containerd.

We don't yet support other container managers (e.g., cri-o).

## Design

For more detailed info about Sysbox's design, refer to the
[Sysboxd design document](docs/design.md).

## OCI Compatibility

Sysboxd is a fork of the [OCI runc](https://github.com/opencontainers/runc). It is mostly
(but not 100%) compatible with the OCI runtime specification. See [here](docs/design.md#oci-compatibility)
for a list of incompatibilities.

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
