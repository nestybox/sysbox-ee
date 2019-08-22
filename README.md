Nestybox Sysvisor
=================

## About Nestybox

Nestybox is re-imagining server virtualization.

We are developing software solutions that improve efficiency,
scalability, performance, and security over current Linux container
and virtualization technologies.

## About Sysvisor

Sysvisor is a container runtime (a.k.a runc) that integrates with
Docker and allows it to create [system containers](docs/system-containers.md).

Sysvisor installs on a Linux host and it's normally not used directly.
Instead, users create containers using Docker and point it to use
the Sysvisor container runtime (see [Usage](#usage) below).

## Features

### Deployment

* Supports deployment of system containers with Docker.

* System containers can run concurrently with regular Docker
  containers, without conflict.

### Security & Isolation

* Strong system container isolation

  - Improves on Docker's container isolation.

  - Strong container-to-host isolation via the Linux user namespace.

  - Strong container-to-container isolation, by using exclusive
    user-ID and group-ID mappings per system container.

* Exposes a partially virtualized procfs (`/proc`) to the container.

  - This makes the container more closely resemble a real host.

  - Prevents processes within the container from changing global kernel
    settings via `/proc`.

### Supported Software

* Supports running Docker inside the system container.

  - Securely, with total isolation between the Docker inside the
    container and the Docker on the host (e.g,. without using Docker
    privileged containers on the host and without bind mounting the
    host's Docker socket into the container).

  - Fast: both the outer Docker and the inner Docker use the overlayfs
    driver.

  - This is useful for testing & CI/CD use cases.

**NOTE**: It's early days for Nestybox, so our system containers support a
reduced set of features (and use-cases) at this time. Please see our
[Roadmap](#roadmap) for a list of features we are working on.

## Supported Linux Distros

* Ubuntu 19.04 "Disco"
* Ubuntu 18.04 "Bionic" (Ubuntu kernel 5.X+ required; see [Host Requirements](#host-requirements) below])

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
  Sysvisor package installer, so there is no need for the user to
  manually type it.

* If the host runs Ubuntu-Bionic, you'll need to update the Linux kernel to
  5.X+ (unless you enable docker [userns-remap](docs/usage.md#interaction-with-docker-userns-remap)).

  Note that you must use the Ubuntu 5.X+ kernel, **not** the Linux upstream kernel.
  The easiest way to do this is to use Ubuntu's [LTS-enablement](https://wiki.ubuntu.com/Kernel/LTSEnablementStack)
  package:

```
$ sudo apt-get update && sudo apt install --install-recommends linux-generic-hwe-18.04 -y
```

## Installation

1) Download the latest package from the [release](https://github.com/nestybox/sysvisor-external/releases) page.

2) Verify that the checksum of the downloaded file fully matches the expected/published one:

```bash
$ sha256sum ~/sysvisor_0.0.1-0~ubuntu-bionic_amd64.deb
2a02898dc53b4751cf413464b977f5b296d9aac3c5b477e05272bfa881d69cfc  /home/user/sysvisor_0.0.1-0~ubuntu-bionic_amd64.deb
```

3) Install the Sysvisor package:

```bash
$ sudo dpkg -i sysvisor_0.0.1-0~ubuntu-bionic_amd64.deb
```

In case you hit an error with missing dependencies, fix this with:

```bash
$ sudo apt-get install -f -y
```

This will install the missing dependencies and automatically re-launch
the Sysvisor installation process.


4) Verify that Sysvisor's systemd units have been properly installed, and
   associated daemons are properly running:

```
$ systemctl list-units -t service --all | grep sysvisor
sysvisor-fs.service                   loaded    active   running Sysvisor-fs component
sysvisor-mgr.service                  loaded    active   running Sysvisor-mgr component
sysvisor.service                      loaded    active   exited  Sysvisor General Service
```

If you are curious on what these Sysvisor services are, refer to the [Sysvisor design document](docs/design.md).

If you hit problems during installation, see the [Troubleshooting document](docs/troubleshoot.md).

## Usage

To launch a system container with Docker, simply use the Docker `--runtime` flag:

```bash
$ docker run --runtime=sysvisor-runc --rm -it --hostname my_cont debian:latest
root@my_cont:/#
```

If you omit the `--runtime` flag, Docker will use the default runc
runtime. It's perfectly fine to run system containers along side with
regular Docker application containers on the host at the same time;
they won't conflict.

Refer to the [Sysvisor User's Guide](docs/usage.md) for other ways to
run system containers with Sysvisor.

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
host's Docker socket into the container. In other words, securely,
with total isolation between the inner and outer Docker containers.

Furthermore, the inner Docker can use the fast overlayfs driver,
rather than the slower vfs driver.

To run Docker-in-Docker, follow the instructions in the [Sysvisor User's Guide](docs/usage.md#docker-in-docker).

## Integration with Container Managers

Sysvisor is designed to work with Docker / containerd.

We don't yet support other container managers (e.g., cri-o).

## Design

For more detailed info about Sysvisor's design, refer to the
[Sysvisor design document](docs/design.md).

## OCI Compatibility

Sysvisor is a fork of the [OCI runc](https://github.com/opencontainers/runc). It is mostly
(but not 100%) compatible with the OCI runtime specification. See [here](docs/design.md#oci-compatibility)
for a list of incompatibilities.

## Production Readiness

Sysvisor is still in Beta. It's stable, but not production ready yet.

## Troubleshooting

Refer to the [Troubleshooting document](docs/troubleshoot.md).

## Issues

We apologize for any problems in the product or documentation, and we appreciate
customers filing issues that help us improve it.

To file issues with Sysvisor (e.g., bugs, feature requests, documentation changes, etc.),
please refer to the [issue guidelines](docs/issue-guidelines.md) document.

## Roadmap

The following is a list of features in the Sysvisor roadmap.

We list these here so that our users can get a better idea of where we
are going and can give us feedback on which of these they like best
(or least).

Nestybox reserves the right to change these based on business
priorities.

Here is the list:

* Support for more Linux distros.

* Support for other container managers (e.g., cri-o)

* Running Kubernetes inside the container

* Running Systemd inside the container

* Running window managers (e.g., X) inside the container (for GUI apps & desktops).

* More virtualization of non-namespaced resources in procfs (`/proc/`).

## Feedback

We love feedback, as it helps us improve Sysvisor and set its future
direction.

We would much appreciate if you would take a couple of minutes to
answer the following survey:

https://www.surveymonkey.com/r/SH8HMGY

## Uninstallation

1) Remove all installed binaries, including 'nbox-shiftfs' kernel submodule:

```bash
$ sudo dpkg --remove sysvisor
```

or,

2) Remove the above items plus all the associated configuration/systemd files (recommended):

```bash
$ sudo dpkg --purge sysvisor
```

## Thank You!

We thank you **very much** for using Sysvisor. We hope you find it useful.

Your trust in us is very much appreciated.

-- *The Nestybox Team*
