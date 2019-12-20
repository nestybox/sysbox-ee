# Sysbox User's Guide

The [Sysbox Quick Start Guide](quickstart.md) contains several examples on how
to install Sysbox and use system containers.

This document supplements the Quick Start Guide with more detailed
information on Sysbox's features, requirements, and restrictions.

## Contents

-   [Running System Containers with Sysbox](#running-system-containers-with-sysbox)
    -   [Using Docker](#using-docker)
    -   [Using the sysbox-runc command](#using-the-sysbox-runc-command)
    -   [Using Other Container Managers](#using-other-container-managers)
-   [System Container Isolation Modes](#system-container-isolation-modes)
    -   [Exclusive userns-remap mode](#exclusive-userns-remap-mode)
    -   [Docker userns-remap mode](#docker-userns-remap-mode)
-   [Running Software inside the System Container](#running-software-inside-the-system-container)
    -   [Docker-in-Docker](#docker-in-docker)
        -   [Inner and Outer Containers](#inner-and-outer-containers)
        -   [Inner Docker Restrictions](#inner-docker-restrictions)
        -   [Inner Docker Image Persistence](#inner-docker-image-persistence)
    -   [Systemd](#systemd)
-   [System Container Bind Mount Requirements](#system-container-bind-mount-requirements)
    -   [Bind Mounts in Exclusive Userns-Remap Mode](#bind-mounts-in-exclusive-userns-remap-mode)
    -   [Bind Mounts in Docker Userns-Remap Mode](#bind-mounts-in-docker-userns-remap-mode)
-   [Storage Sharing Among System Containers](#storage-sharing-among-system-containers)
-   [Support for Linux Security Modules](#support-for-linux-security-modules)
    -   [AppArmor](#apparmor)
    -   [SELinux](#selinux)
    -   [Other LSMs](#other-lsms)
-   [Support for Exposing Host Devices inside System Containers](#support-for-exposing-host-devices-inside-system-containers)
-   [Support for `docker run userns`](#support-for-docker-run-userns)
-   [Rootless Container Support](#rootless-container-support)
-   [Privileged Container Support](#privileged-container-support)
-   [Checkpoint and Restore Support](#checkpoint-and-restore-support)
-   [Sysbox Reconfiguration](#sysbox-reconfiguration)

## Running System Containers with Sysbox

We currently support two ways of running system containers with Sysbox:

1) Using Docker (the easy and preferred way)

2) Using the `sysbox-runc` command directly (more control but harder)

Both of these are explained below.

### Using Docker

It's easy to run system container using Docker. Simply add the `--runtime=sysbox-runc`
flag in the `docker run` command:

```console
$ docker run --runtime=sysbox-runc --rm -it --hostname my_cont debian:latest
root@my_cont:/#
```

If you wish, you can configure Sysbox as the default runtime for Docker. This
way you don't have to use the `--runtime` flag every time. To do this,
refer to this [Docker article](https://docs.docker.com/engine/reference/commandline/dockerd/).

### Using the sysbox-runc command

It's also possible to launch system containers directly via the
`sysbox-runc` command. This is useful when wishing to control
the configuration of the system container at the lowest level.

As the root user, follow these steps:

1) Create a rootfs image for the system container:

```console
# mkdir /root/syscontainer
# cd /root/syscontainer
# mkdir rootfs
# docker export $(docker create debian:latest) | tar -C rootfs -xvf -
```

2) Create the OCI spec (i.e., `config.json` file) for the system container:

```console
# sysbox-runc spec
```

3) Launch the system container.

Choose a unique name for the container and run:

```console
# sysbox-runc run my_syscontainer
```

Use `sysbox-runc --help` command for help on all commands supported.

Also, in step (2) above, feel free to modify the system container's
`config.json` to your needs. But note that Sysbox ignores a few
of the OCI directives in this file (refer to the [Sysbox design document](design.md#oci-compatibility)
for details).

### Using Other Container Managers

We officially only support the above methods to run Sysbox.

However, we plan to add support for other OCI compatible container
managers (e.g., [cri-o](https://cri-o.io/)) soon.

## System Container Isolation Modes

Nestybox system containers **always** use the Linux user-namespace for
enhanced isolation between the system container processes and the rest
of the system.

This is one key thing that differentiates them from regular Docker
containers, and it's done in order for the system container to be more
strongly isolated from the rest of the system while enabling
functionality that requires root in the system container to have full
privileges within the container.

The Linux user namespace works by mapping user-IDs and group-IDs
between the container and the host. There are different ways this
mapping can be done, which give rise to what we call "system container
isolation modes".

Sysbox supports the following system container isolation modes, listed
from strongest to weakest isolation.

-   [Exclusive userns-remap mode](#exclusive-userns-remap-mode)
-   [Docker userns-remap mode](#docker-userns-remap-mode)

A quick summary of the modes is shown in the table below.

| Isolation Mode         | User Namespace User-ID and Group-ID Mappings | Isolation Strength | Enabling Action                               | Shiftfs |
| ---------------------- | -------------------------------------------- | ------------------ | --------------------------------------------- | ------- |
| Exclusive userns-remap | Exclusive per system container               | Highest            | None (default operating mode)                 | Yes     |
| Docker userns-remap    | Common for all system containers             | Medium             | Configure the Docker daemon with userns-remap | No      |

More detailed explanations on each isolation mode and how to enable
them are in the sub-sections that follow.

### Exclusive userns-remap mode

In this mode, Sysbox allocates **exclusive** user-namespace user-ID and
group-ID mappings to each system container.

This ensures that if a system container process somehow escapes the
container's root filesystem jail, it will find itself without any
permissions to access any other files in the system.

It's the most secure mode offered by Sysbox, and it's the default
operating mode for Nestybox system containers (whether they are
deployed with Docker or directly via the sysbox-runc command). No
action from the user is required to enable this mode.

Note that this mode requires the presence of the [shiftfs module](design.md#ubuntu-shiftfs-module)
in the Linux kernel, which in turn restricts the
[distros](distro-compat.md) on which Sysbox is supported.

For further details on Sysbox's usage of the Linux user namespace and
exclusive user-ID/group-ID mappings, refer to the [Sysbox design document](design.md#linux-namespace-usage).

### Docker userns-remap mode

This mode is automatically entered by Sysbox when the Docker daemon
is configured with "userns-remap" enabled, as described in this
[Docker article](https://docs.docker.com/engine/security/userns-remap).

When Docker is configured with userns-remap, it enables the
user-namespace in all containers and configures the user-ID and
group-ID mappings for them. Sysbox automatically detects this and
honors Docker's selection of user-ID and group-ID mappings.

Note however that this mode is less secure than the exclusive
userns-remap mode described previously because Docker currently uses
the same user-ID and group-ID mappings for all containers, thereby
decreasing cross-container isolation (i.e., root in all containers
maps to the same user-ID and group-ID in the host).

Another caveat with Docker userns-remap mode is that it causes Docker
to enable the user-namespace in all containers, regardless of which
runtime they are spawned with (e.g., Docker's runc or sysbox-runc).
This imposes a few limitations on regular Docker containers, as
described at the end of this [Docker document](https://docs.docker.com/engine/security/userns-remap).

In contrast, in exclusive userns-remap mode only containers spawned
with sysbox-runc use the user-namespace, while containers spawned with
Docker's default runc do not.

Having said this, this mode is useful because it's a bit more mature
than the [Exclusive userns-remap mode](#exclusive-userns-remap-mode)
and does not require the presence of the shiftfs module in the Linux
kernel, thereby increasing the [distros](distro-compat.md)
on which it's supported.

## Running Software inside the System Container

A system container is logically a super-set of a regular Docker
application container and thus should be able to run any application
that runs in a regular Docker container, plus system-level software
(e.g., Docker inside the system container).

Nestybox's goal is to allow you run any software inside the system
container just as you would on a physical host or VM.

The sub-sections below provide information on running system-level
software inside a system container. This software does not normally
run inside a regular Docker container (unless unsecure privileged
containers are used and/or complex container configurations are set).

### Docker-in-Docker

Nestybox system containers support running Docker inside the system
container, without using privileged containers or bind-mounting the
host's Docker socket into the container. In other words, cleanly and
securely, with **total isolation** between the inner and outer Docker
daemons.

This is useful for Docker sandboxing, testing and CI/CD use cases.

Moreover, it's fast: the Docker daemon inside the container uses the
fast overlay2 (or btrfs) storage drivers, rather than alternative
Docker-in-Docker solutions that resort to the slower vfs driver.

To run Docker inside a system container (a.k.a Docker-in-Docker), the
easiest way is to use a system container image that has Docker
pre-installed in it.

You can find a few such images in the [Nestybox DockerHub repo](https://hub.docker.com/r/nestybox).

The [Sysbox Quickstart Guide](quickstart.md) has several examples showing how
to run Docker inside a system container.

#### Inner and Outer Containers

When launching Docker inside a system container, terminology can
quickly get confusing due to container nesting.

To prevent confusion we refer to the containers as the "outer" and
"inner" containers.

-   The outer container is a system container, created at the host
    level; it's launched with Docker + Sysbox.

-   The inner container is an application container, created within the
    outer container (i.e., it's created by the Docker instance running
    inside the system container (aka the inner Docker)).

#### Inner Docker Restrictions

The Docker instance inside the system container is assumed to store
it's images at the usual `/var/lib/docker`. This directory is known as
the Docker "data-root".

While it's possible to configure the inner Docker to store it's images
at some other location within the system container (via the Docker
daemon's `--data-root` option), Sysbox does **not** currently support
this (i.e., the inner Docker won't work).

#### Inner Docker Image Persistence

The Docker instance running inside the system container stores its
images in the `/var/lib/docker` directory inside the container.

When the system container is removed (i.e., not just stopped, but
actually removed via `docker rm`), the contents of that directory will
also be removed. In other words, the inner Docker's image cache is
destroyed when the associated system container is removed.

It's possible to override this behavior by mounting a host directory
into the system container's `/var/lib/docker`, in order to persist the
inner Docker's image cache across system container life-cycles.

The Sysbox Quick Start Guide has examples [here](quickstart.md#persistence-of-inner-container-images-with-docker-volumes)
and [here](quickstart.md#persistence-of-inner-container-images-with-bind-mounts).

A warning though: a given host directory mounted into a system
container's `/var/lib/docker` must only be mounted on a **single
system container at any given time**. This is a restriction imposed by
the Docker daemon, which does not allow its image cache to be shared
concurrently among multiple daemon instances. Sysbox will check
for violations of this rule and report an appropriate error during
system container creation.

### Systemd

Nestybox has preliminary support for running Systemd inside a system
container, meaning that Systemd works but there are still some minor
issues that need resolution.

Deploying Systemd inside a system container is useful when you plan to
run multiple services inside the system container, or when you want to
use it as a virtual host environment.

Unlike other solutions, Nestybox system containers run Systemd securely
(without resorting to privileged Docker containers) and without burden
on the user: simply launch a system container image with Systemd as
its entry point and Sysbox will ensure the system container is setup
to run Systemd without problems.

The [Sysbox Quick Start Guide](quickstart.md) has a few examples of this.

## System Container Bind Mount Requirements

Sysbox system containers support all Docker storage mount types:
[volume, bind, or tmpfs](https://docs.docker.com/storage/).

However, for bind mounts there are important requirements in order to
deal with file ownership and security issues.
These requirements vary depending on the [system container isolation mode](#system-container-isolation-modes).

The following table has a quick summary of the bind mount requirements:

| System Container Isolation Mode | Bind Source Ownership on Host | Recommended Security Precautions                            |
| ------------------------------- | ----------------------------- | ----------------------------------------------------------- |
| Exclusive userns-remap          | 0 to 65535                    | Bind source accessible by root only.                        |
|                                 |                               | Mount as read-only in sys container when possible.          |
|                                 |                               | Remount on host as `noexec` prior to running sys container. |
| Docker userns-remap             | uid to (uid+65535)            | None                                                        |
|                                 | (where `uid` is the the subid |                                                             |
|                                 | associated with the Docker    |                                                             |
|                                 | userns remap configuration)   |                                                             |

The sub-sections below describe in detail the rationale for these
requirements and provide examples.

Note that the requirements listed in the table above are not specific
Nestybox system containers; these are also generally applicable to
bind mounts on any Docker container.

Finally, the requirements don't apply to Docker volume or tmpfs
mounts, as they are implicitly met by these.

### Bind Mounts in Exclusive Userns-Remap Mode

When a system container that uses [exclusive userns-remap mode](#exclusive-userns-remap-mode)
is started with a bind mount (e.g., `docker run --runtime=sysbox-runc --mount type=bind,source=/some/source,target=/some/target ...`),
Sysbox

-   Sysbox automatically mounts the Ubuntu shiftfs filesystem on the
    bind-mount source when the container starts; the shiftfs mount is kept
    in place until the container is stopped.

-   Sysbox uses the shiftfs mount to ensure that processes inside the
    system container see the correct file ownership for the contents of
    the bind mounted directory, as described [here](design.md#ubuntu-shiftfs-module).

As a result, the following requirements apply to system container bind
mounts:

-   The bind mounted source directory or file should be owned by a user-ID
    and group-ID in the range [0:65535]. The used-ID and group-ID of
    these files will be mapped inside the system container as follows:

    | File User/Group-ID on Host | File User/Group-ID in System Container |
    | -------------------------- | -------------------------------------- |
    | 0 to 65535                 | 0 to 65535                             |
    | Others                     | nobody:nogroup                         |

    In other words, files owned by root in the bind-mounted source
    directory are owned by the root user in the system container. Files
    owned by user 1000 in the bind-mounted source directory are owned by
    user 1000 inside the system container. And so on.

    Note that it's possible for multiple system containers to share a
    bind-mount source, as all system containers will use the file
    ownership mapping shown above. The Sysbox Quick Start Guide has
    an example of this [here](quickstart.md#sharing-storage-among-system-containers).

-   For environments where security is important, we recommend that one
    or more of the following actions be applied to the bind mount
    source file or directory:

    -   Make it accessible only by host's root user (i.e., `0700` permission
        somewhere in the mount source's absolute path).

    -   Mount it into the system container as "read-only" when possible. For example:

    ```console
    $ docker run --runtime=sysbox-runc -it --mount type=bind,source=/path/to/bind/source,target=/path/to/mnt/point,readonly my-syscont
    ```

    -   Re-mount it on the host with the `noexec` attribute prior to
        bind mounting it into the system container. For example:

    ```console
    $ sudo mount --bind /path/to/bind/source /path/to/bind/source
    $ sudo mount -o remount,bind,noexec /path/to/bind/source /path/to/bind/source
    $ docker run --runtime=sysbox-runc -it --mount type=bind,source=/path/to/bind/source,target=/path/to/mnt/point my-syscont`
    ```

The above requirements are not "hard requirements", meaning that
Sysbox won't check for them (i.e., if they aren't met the system
container will still work). However, failure to meet one of these
requirements will result in incorrect file ownership on the bind mount
or reduced host security as described [here](design.md#shiftfs-security-precautions).

### Bind Mounts in Docker Userns-Remap Mode

When a system container is started with [Docker userns-remap mode](#docker-userns-remap-mode), the
following rule applies to bind mounts:

-   The bind mounted directory or file should be owned by users in the
    range [uid:uid+65535], where `uid` is the subid range associated
    with the host's user-ID set in the docker userns-remap
    configuration. The same applies to group-IDs.

For example, if the docker userns-remap configuration is the following:

```console
$ cat /etc/docker/daemon.json
{
   "userns-remap": "someuser"
}
```

then the bind mount source directory and/or files must be owned by the
subuid(gid) range associated with user `someuser`. That range can be
found in the host's `/etc/subuid` and `/etc/subgid` files.

This rule ensures that files show up with appropriate ownership inside
the system container.

For example, assume the following configuration of subuids(gids):

```console
$ cat /etc/subuid
someuser:100000:65536
sysbox:165536:268435456

$ cat /etc/subgid
someuser:100000:65536
sysbox:165536:268435456
```

Then the bind mount source should be owned by `user-ID:group-ID` =
`100000:100000` because that's the start of the subuid(gid) range
associated with `someuser`.

Note that the security requirements listed in the [prior section](#bind-mounts-in-exclusive-userns-remap-mode)
don't apply when Docker is configured with userns-remap. That's
because files written from within the system container will have
ownership corresponding to the subuid(gid) associated with user
`someuser` rather than `root:root` and thus don't present a security
risk on the host.

## Storage Sharing Among System Containers

System containers use the Linux user-namespace for increased isolation
from the host and from other containers.

A known issue with containers that use the user-namespace is that
sharing storage between them is not trivial because each container may
be assigned an exclusive user-ID(group-ID) range on the host, and thus
may not have permissions to access the files in the shared storage
(unless such storage has lax permissions allowing read-write-execute
by any user).

Sysbox system containers support storage sharing between multiple
system containers, without lax permissions and in spite of the fact
that each system container may be assigned an exclusive
user-ID/group-ID range on the host.

In order to share storage between multiple system containers, simply
run the system containers and bind mount the storage into them.

When performing the bind-mount, the requirements described in
section [System Container Bind Mount Requirements](#system-container-bind-mount-requirements)
apply.

The Sysbox Quick Start Guide has an example of multiple system containers
sharing storage [here](#storage-sharing-among-system-containers)).

## Support for Linux Security Modules

### AppArmor

Sysbox currently uses a permissive AppArmor profile for system
containers. We plan to create a more restrictive profile in
the near future.

### SELinux

Sysbox does not yet support running on systems with SELinux enabled.

### Other LSMs

Sysbox does not have support for other Linux LSMs at this time.

## Support for Exposing Host Devices inside System Containers

Sysbox does not currently support exposing host devices inside system
containers (e.g., via the `docker run --device` option). We are
working on adding support for this.

## Support for `docker run userns`

When Docker is configured in userns-remap mode, Docker offers the ability
to disable that mode on a per container basis via the `--userns=host`
option in the `docker run` and `docker create` commands.

This option **does not work** with Sysbox (i.e., don't use
`docker run --userns=host --runtime=sysbox-runc ...`).

Usage of this option is rare as it can lead to the problems as
described [in this Docker article](https://docs.docker.com/engine/security/userns-remap/#disable-namespace-remapping-for-a-container).

## Support for Read-Only Root Filesystem

Sysbox does not currently support system containers configured with a
read-only rootfile system (e.g., those created with the `docker run
--read-only` flag).

## Docker cgroup driver restriction

Docker uses the Linux cgroups feature to place resource limits on containers.

Docker supports two cgroup drivers: `cgroupfs` and `systemd`.
In the former, Docker directly manages the container resource
limits. In the latter, Docker works with Systemd to manage the
resource limits. The `cgroupfs` driver is the default driver.

Sysbox currently requires that Docker be configured with the
`cgroupfs` driver. Sysbox does not currently work when Docker is
configured with the `systemd` cgroup driver.

## Out-of-Memory Score Adjustment

The Linux kernel has a mechanism to kill processes when the
system is running low on memory.

The decision on which process to kill is done based on an
out-of-memory (OOM) score assigned to all processes. The score is in
the range [-1000:1000], where higher values means higher probability
of the process being killed when the host reaches an out-of-memory
scenario.

It's possible for users with sufficient privileges to adjust the OOM
score of a given process, via the `/proc/[pid]/oom_score_adj` file.

A system container's init process OOM score adjustment can be
configured to start at a given value. For example, when using Docker
to deploy the system container, by passing the `--oom-score-adj`:

```console
$ docker run --runtime=sysbox-runc --oom-score-adj=-100 -it alpine:latest
```

In addition, Sysbox ensures that system container processes are
allowed to modify their out-of-memory (OOM) score adjustment to any
value in the range [-999:1000], via `/proc/[pid]/oom_score_adj`. This
is necessary in order to allow system software that requires such
adjustments to operate correctly within the system container.

From a host security perspective however, allowing system container
processes to adjust their OOM score downwards is risky, since it
means that such processes are unlikely to be killed when the host is
running low on memory.

To mitigate this risk, a user can always put an upper bound on the
memory allocated to each system container. Though this does not
prevent a system container process from reducing its OOM score,
placing such an upper bound reduces the chances of the system
running out of memory and prevents memory exhaustion attacks
by malicious processes inside the system container.

Placing an upper bound on the memory allocated to the system container
can be done by using Docker's `--memory` option:

```console
$ docker run --runtime=sysbox-runc --memory=100m -it alpine:latest
```

## Rootless Container Support

Sysbox must run with root privileges on the host system. It won't
work if executed without root privileges.

## Privileged Container Support

System containers must never be launched using the Docker
`--privileged` flag (in fact doing so will fail). Recall that one key
reason for using system containers is to use them as a secure
alternative to privileged containers (i.e., one that can run system
level workloads but with enhanced isolation from the underlying host).

Within a system container however, you *can* deploy privileged Docker
containers (e.g., by installing Docker inside the system container
and deploying inner containers with the `docker run --privileged`
flag).

Note however that a privileged container inside a system container is
privileged within the context of the system container only, but *not*
on the underlying host. In other words, such a privileged container
can access all resources of the parent system container, but not other
system resources. For example, when running a privileged container
inside a system container, the `/proc` mounted inside the privileged
container does not allow access to all host resources. Rather, it only
allows access to resources associated with the system container.

The ability to run privileged containers inside a system container is
useful when using the system container as a Docker sandbox and
deploying inner containers that require full privileges (typically
containers containing system services).

## Checkpoint and Restore Support

Sysbox does not currently support checkpoint and restore support of
system containers.

## Sysbox Reconfiguration

The Sysbox installer starts the [Sysbox components](design.md#sysbox-components)
automatically, using the host's Systemd.

Normally this is sufficient and the user need not worry about re-configuring Sysbox.

However, there are scenarios where the daemons may need to be
reconfigured (e.g., to enable a given option on sysbox-fs or
sysbox-mgr). For example, increasing the log level by passing the
`--log-level debug` to the sysbox-fs or sysbox-mgr daemons.

In order to reconfigure Sysbox, do the following:

1) Stop all system containers (there is a sample script for this [here](../scr/rm_all_syscont)).

2) Modify the desired service initialization instruction.

   For example, if you wish to change the log-level, do the following:

```console
$ sudo sed -i --follow-symlinks '/^ExecStart/ s/$/ --log-level debug/' /lib/systemd/system/sysbox-fs.service
$
$ egrep "ExecStart" /lib/systemd/system/sysbox-fs.service
ExecStart=/usr/local/sbin/sysbox-fs --log /var/log/sysbox-fs.log --log-level debug
```

3) Reload Systemd to digest the previous change:

```console
$ sudo systemctl daemon-reload
```

4) Restart the sysbox service:

```console
$ sudo systemctl restart sysbox
```

5) Verify the sysbox service is running:

```console
$ sudo systemctl status sysbox.service
● sysbox.service - Sysbox General Service
   Loaded: loaded (/lib/systemd/system/sysbox.service; enabled; vendor preset: enabled)
   Active: active (exited) since Sun 2019-10-27 05:18:59 UTC; 14s ago
  Process: 26065 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
 Main PID: 26065 (code=exited, status=0/SUCCESS)

Oct 27 05:18:59 disco1 systemd[1]: sysbox.service: Succeeded.
Oct 27 05:18:59 disco1 systemd[1]: Stopped Sysbox General Service.
Oct 27 05:18:59 disco1 systemd[1]: Stopping Sysbox General Service...
Oct 27 05:18:59 disco1 systemd[1]: Starting Sysbox General Service...
Oct 27 05:18:59 disco1 systemd[1]: Started Sysbox General Service.
```

That's it. You can now launch system containers.

Note that even though Sysbox is comprised of various daemons and its
respective services, you should only interact with its outer-most
systemd service called "sysbox".
