Nestybox Sysvisor
=================

## Introduction

Sysvisor is a container runtime that enables creation of system
containers.

A system container is a container whose main purpose is to package and
deploy a full operating system environment.

Sysvisor system containers use all Linux namespaces (including the
user-namespace) and cgroups for strong isolation from the underlying
host and other containers.


## Key Features

* Designed to integrate with Docker

  - Users launch and manage system containers with Docker (i.e.,
    `docker run ...`), just as with any other container.

  - Leverages Docker's container image caching & sharing technology
    (storage efficient).

* Strong container-to-host isolation via use of *all* Linux namespaces

  - Including the user namespace (i.e., root in the system container
    maps to a non-root `nobody:nogroup` user in the host).

* Supports Docker daemon configured with `userns-remap` or without it.

  - With `userns-remap`, Docker manages the container's user
    namespace; however, Docker currently assigns all containers the
    same user-id/group-id mapping, reducing container-to-container
    isolation.

  - Without userns-remap, Sysvisor manages the container's user
    namespace; Sysvisor assigns *exclusive* user-id/group-id mappings
    to each container, for improved container-to-container isolation.

* Supports running Docker *inside* the system container.

  - Securely, without using Docker privileged containers.

* Supports shared storage between system containers

  - Without requiring lax permissions on the shared storage.

The following sections describe how to use each of these features.


## Sysvisor Components

Sysvisor is made up of the following components:

* sysvisor-runc

* sysvisor-fs

* sysvisor-mgr


sysvisor-runc is the program that creates containers. It's the
frontend of sysvisor: higher layers (e.g., Docker) invoke
sysvisor-runc to launch system containers.

sysvisor-fs is a file-system-in-user-space (FUSE) that emulates
portions of the container's filesystem, in particular the `/proc/sys`
directory. It's purpose is to expose system resources inside the
system container with proper isolation from the host.

sysvisor-mgr is a daemon that provides services to sysvisor-runc and
sysvisor-fs. For example, it manages assignment of exclusive user
namespace subuid mappings to system containers.

Together, sysvisor-fs and sysvisor-mgr are the backends for
sysvisor. Communication between sysvisor components is done via gRPC.


## Supported Linux Distros

When using Sysvisor with Docker userns-remap:

* Ubuntu 18.04 (Bionic)
* Ubuntu 18.10 (Cosmic)
* Ubuntu 19.04 (Disco)

When using Sysvisor without Docker userns-remap:

* Ubuntu 19.04 (Disco)


## Host Requirements

* Docker must be installed on the host machine.

* The host's kernel should be configured to allow unprivileged users
  to create namespaces. This config likely varies by distro; thereby,
  we expect an official kernel from a supported linux distribution
  to be utilized on the host machine.

  Ubuntu's example:

  ```
  sudo sh -c "echo 1 > /proc/sys/kernel/unprivileged_userns_clone"
  ```

    Note: The above instruction will be *automatically* executed by the
    package installer, so there is no need for the user to manually
    type it.

* Systemd: The host should be running systemd as the system's process-manager.
This is usually the case as most well-known Linux distributions (including
Sysvisor's supported ones) rely on systemd as their default process-manager.


## Installation

* Download the latest package from the [release](https://github.com/nestybox/sysvisor-external/releases) page.


* Verify that the checksum of the downloaded file fully matches the expected/published one:

    ```bash
    $ sha256sum ~/sysvisor_0.0.1-0~ubuntu-bionic_amd64.deb
    2a02898dc53b4751cf413464b977f5b296d9aac3c5b477e05272bfa881d69cfc  /home/user/sysvisor_0.0.1-0~ubuntu-bionic_amd64.deb
    ```

* Install Sysvisor package:

    ```
    $ sudo dpkg -i sysvisor_0.0.1-0~ubuntu-bionic_amd64.deb
    ```

    Expected output:

    ```
    Selecting previously unselected package sysvisor.
    (Reading database ... 150254 files and directories currently installed.)
    Preparing to unpack .../sysvisor_0.0.1-0~ubuntu-bionic_amd64.deb ...
    Unpacking sysvisor (1:0.0.1-0~ubuntu-bionic) ...
    Setting up sysvisor (1:0.0.1-0~ubuntu-bionic) ...

    Disruptive changes made to docker configuration. Restarting docker service...
    Created symlink /etc/systemd/system/sysvisor.service.wants/sysvisor-fs.service → /lib/systemd/system/sysvisor-fs.service.
    Created symlink /etc/systemd/system/sysvisor.service.wants/sysvisor-mgr.service → /lib/systemd/system/sysvisor-mgr.service.
    Created symlink /etc/systemd/system/multi-user.target.wants/sysvisor.service → /lib/systemd/system/sysvisor.service.
    ```

* In case an error is observed above as a consequence of a missing
software dependency, proceed to download/install this package as
indicated below. Once this requirement is satisfied, Sysvisor's
installation process will be automatically re-launched to conclude
this task.

    Missing dependency output:

    ```
    ...
    dpkg: dependency problems prevent configuration of sysvisor:
    sysvisor depends on jq; however:
    Package jq is not installed.

    dpkg: error processing package sysvisor (--install):
    dependency problems - leaving unconfigured
    Errors were encountered while processing:
    sysvisor
    ```

    Install missing package by fixing (-f) system's dependency
    structures.

    ```
    $ sudo apt-get install -f -y
    ```

* Verify that Sysvisor's systemd units have been properly installed, and
associated daemons are properly running:

    ```
    $ systemctl list-units -t service --all | grep sysvisor
    sysvisor-fs.service                   loaded    active   running Sysvisor-fs component
    sysvisor-mgr.service                  loaded    active   running Sysvisor-mgr component
    sysvisor.service                      loaded    active   exited  Sysvisor General Service
    ```


## Usage -- host context

* System-container creation

    We will make use of docker's existing `--runtime` cli attribute
    during the creation of a system-container. This element shall be
    defined as per the example shown below. Other than that, feel
    free to make use of the docker-cli attributes that you are
    familiarized with.

    The example below creates a system-container with 'syscont' as
    its hostname, and making use of 'debian:latest' as its baseline
    image:

    ```
    $ docker run --runtime=sysvisor-runc --rm -it --hostname syscont debian:latest
    root@syscont:/#
    ```

* System-container logs

    Sysvisor's daemons (i.e. sysvisor-fs and sysvisor-mgr) will log
    information related to system-container's activities in
    `/var/log/sysvisor-fs.log` and `/var/log/sysvisor-mgr.log` file
    respectively. These logs should be useful during troubleshooting
    exercises.


## Usage -- system-container context


### Supported apps

* Docker

  - May be installed inside a system container using the instructions
    in the Docker website.

  - Alternatively, use a system container image that containers a
    pre-installed Dockerd (e.g.,
    `nestybox/sys-container:debian-plus-Docker`)

