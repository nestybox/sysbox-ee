Nestybox Sysvisor
=================

## About Nestybox

Nestybox helps application developers deploy containers in ways that
extend beyond packaging a single application, and with additional
security.

## About Sysvisor

Sysvisor is a container runtime (a.k.a runc) that integrates with
Docker and allows it to run containers that enhance regular Docker
containers in two ways:

* It expands the types of programs that can run within the container.

* It hardens the isolation of the container from the rest of the
  system.

## Key Features

* Improves Docker container isolation from the host and other containers:

  - Uses all Linux namespaces.

  - Uses exclusive user-ID and group-ID mappings per container.

* Supports running Docker *inside* the container (i.e., docker-in-docker)

  - Securely, with total isolation between the Docker inside the container
    and host (e.g,. without using Docker privileged containers on the
    host and without bind mounting the host's Docker sockets).

  - This is useful for testing & CI/CD use cases.

  - It also improves application security by adding another layer of
    isolation between the application and the host.

* Exposes a partially virtualized procfs (`/proc`) to the container.

  - This makes the container more closely resemble a real host, and
    further improves isolation from the rest of the system.

* Supports running side-by-side with the default Docker runc.

  - This allows users to run regular Docker containers side-by-side
    with Sysvisor enhanced containers, without any conflict.

## Supported Linux Distros

When using Sysvisor without Docker userns-remap (for strongest
container isolation):

* Ubuntu 19.04 (Disco)

When using Sysvisor with Docker userns-remap:

* Ubuntu 18.04 (Bionic)
* Ubuntu 18.10 (Cosmic)
* Ubuntu 19.04 (Disco)

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

To launch a container with Docker + Sysvisor, simply use the Docker `--runtime` flag:

```bash
$ docker run --runtime=sysvisor-runc --rm -it --hostname my_cont debian:latest
root@my_cont:/#
```

If you omit the `--runtime` flag, Docker will use the default runc
runtime. It's perfectly fine to have Sysvisor enhanced containers
running along side with regular Docker containers on the host at the
same time; they won't conflict.

Refer to the [Sysvisor User's Guide](docs/usage.md)
for other ways to run Sysvisor containers.

## Applications supported inside the container

A Sysvisor container is an enhanced Docker container, and should be
able to run any application that runs in a regular Docker container,
plus some additional ones (e.g., Docker).

### Docker-in-Docker

To run Docker inside a Sysvisor container (e.g., Docker-in-Docker),
launch the container and then install Docker using the instructions in
the Docker website.

Once Docker is installed inside the container, launch the Docker
daemon with:

```bash
root@my_cont:/# dockerd &
```

And then run the (inner) containers as usual:

```bash
root@my_cont:/# docker run -it --hostname my_inner_cont busybox
root@my_inner_cont:/#
```

A better way to do the above is to create a Dockerfile that contains
those same Docker installation instructions, in order to create a
container image that has Docker pre-installed in it.

There is a sample Dockerfile [here](dockerfiles/dind/Dockerfile).
Feel free to use it and modify it to your needs.

### Inner & Outer Containers

When launching Docker inside a container, terminology can quickly get
confusing due to container nesting.

To prevent confusion we refer to the containers as the "outer" and
"inner" containers.

* The outer container is created at the host level; it's launched with
  Docker + Sysvisor.

* The inner container is created from within the outer container. It's
  launched by the Docker instance running inside the outer container.

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

We apologize for any problems in the product, and we appreciate
customers filing issues that help us improve it.

For filing issues with Sysvisor (e.g., bugs, feature requests, documentation changes, etc.),
please refer to the [bug filing guidelines](docs/bug-filing.md) document.

## Roadmap

The following is a list of features in the Sysvisor roadmap.

We list these here so that our users can get a better idea of where we
are going and can give us feedback on which of these they like best
(or least).

Nestybox reserves the right to change these based on business
priorities.

Here is the list:

* Running Kubernetes inside the container

* Running Systemd inside the container

* Running window managers (e.g., X) inside the container (for GUI).

* More virtualization of non-namespaced resources under `/proc/`.

* Support for other container managers (e.g., cri-o)

## Feedback

We love feedback (especially constructive one), as it helps us improve Sysvisor.

Do tell us please:

* What's your use case for Sysvisor?

* What current features you like most?

* What features you would like to see?

* What features in our roadmap you like best (or least)?

* What features don't work well and how can we improve them?

This is valuable information for us. While we can't guarantee that we
will implement all requests, we will definitely listen and do our best
to implement the requests that are most compelling to our users.

TODO: what's the process for them to tell us?

## Thank You!

We thank you **very much** for using Sysvisor. We hope you find it useful.

Our mission is to help you use Docker in new ways to solve your
container-related problems.

Your trust in us is very much appreciated.

-- *The Nestybox Team*
