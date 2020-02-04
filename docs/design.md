# Sysbox Design Notes

This document briefly describes some aspects of Sysbox's design.

## Contents

-   [Sysbox Components](#sysbox-components)
-   [Linux Namespace Usage](#linux-namespace-usage)
    -   [User Namespace](#user-namespace)
    -   [Cgroup Namespace](#cgroup-namespace)
-   [Exclusive User Namespace Mappings](#exclusive-user-namespace-mappings)
-   [Ubuntu Shiftfs Module](#ubuntu-shiftfs-module)
    -   [Shiftfs Security Precautions](#shiftfs-security-precautions)
    -   [Shiftfs Functional Limitations](#shiftfs-functional-limitations)
-   [Procfs Virtualization](#procfs-virtualization)
-   [OCI compatibility](#oci-compatibility)
    -   [Namespaces](#namespaces)
    -   [Process Capabilities](#process-capabilities)
    -   [Procfs](#procfs)
    -   [Cgroupfs Mount](#cgroupfs-mount)
    -   [Seccomp](#seccomp)
    -   [AppArmor](#apparmor)
    -   [Read-only and Masked Paths](#read-only-and-masked-paths)
    -   [Mounts](#mounts)
-   [Sysbox Nesting](#sysbox-nesting)

## Sysbox Components

Sysbox is made up of the following components:

-   sysbox-runc

-   sysbox-fs

-   sysbox-mgr

sysbox-runc is a container runtime, the program that does the low
level kernel setup for execution of system containers. It's the
"front-end" of sysbox: higher layers (e.g., Docker & containerd)
invoke sysbox-runc to launch system containers. It's mostly (but not
100%) compatible with the OCI runtime specification (more on this
[here](#oci-compatibility)).

sysbox-fs is a file-system-in-user-space (FUSE) daemon that emulates
portions of the system container's filesystem, in particular portions
of the `/proc` directory. It's purpose is to make the system container
more closely resemble a real host while ensuring proper isolation from
the host.

sysbox-mgr is a daemon that provides services to sysbox-runc and
sysbox-fs. For example, it manages assignment of exclusive user
namespace user-ID and group-ID mappings to system containers.

Together, sysbox-fs and sysbox-mgr are the "back-ends" for
sysbox. Communication between the sysbox components is done via
gRPC.

Users don't normally interact with the Sysbox components directly.
Instead, they use higher level apps (e.g., Docker) that interact with
Sysbox to deploy system containers.

## Linux Namespace Usage

System containers deployed with Sysbox always use _all_ Linux
namespaces for enhanced isolation & security from the rest of the
system.

That is, when you deploy a system container with Docker + Sysbox
(e.g., `docker run --runtime=sysbox-runc -it alpine:latest`), Sysbox
will always setup the container with all Linux namespaces enabled.

This is one area where Sysbox deviates from the OCI specification,
which leaves it to the higher layer (e.g., Docker + containerd) to
choose the namespaces that should be enabled for the container.

In addition to providing enhanced isolation, using all Linux namespaces
(specially the user namespace) allows the system container to run more
types of workloads, in particular system-level workloads that require
root inside the container to have many capabilities within the
container.

The table below shows a comparison on namespace usage between
Nestybox system containers and regular Docker containers.

| Namespace | Docker + Sysbox | Docker + runC                                                              |
| --------- | --------------- | -------------------------------------------------------------------------- |
| mount     | Yes             | Yes                                                                        |
| pid       | Yes             | Yes                                                                        |
| uts       | Yes             | Yes                                                                        |
| net       | Yes             | Yes                                                                        |
| ipc       | Yes             | Yes                                                                        |
| cgroup    | Yes             | No                                                                         |
| user      | Yes             | No by default; Yes when Docker engine is configured with userns-remap mode |

### User Namespace

Nestybox system containers always use the Linux user-namespace. This
is a key feature for enhanced isolation, as it confines the privileges
of processes running inside the container to resources assigned to the
container.

For example, a process with full capabilities inside the container
(root inside the container) is only capable of using those
capabilities on resources assigned to the container itself. It can't
access resources not assigned to the container (e.g., global system
resources, resources assigned to other containers, etc.)

### Cgroup Namespace

In addition, system containers also use the cgroup namespace, which
virtualizes cgroup information exposed via the system container's
`/proc` filesystem. The end result is that it hides host paths related
to the system container's cgroups from processes inside the system
container. Refer to the kernel's
[cgroup_namespaces][cgroup-namespaces] manual page for more info.

## Exclusive User Namespace Mappings

The Linux user namespace works by mapping user-IDs and group-IDs
between the container and the host (or more precisely between the
container's user namespace and its parent user namespace).

There are different ways this mapping can be done, which give rise to
the "system container isolation modes".

As described in the [user guide](usage.md), Sysbox supports the
following system container isolation modes.

-   [Exclusive userns-remap mode](usage.md#exclusive-userns-remap-mode)
-   [Docker userns-remap mode](usage.md#docker-userns-remap-mode)

In this section, we will focus on exclusive userns-remap mode and the
manner in which Sysbox allocates user-ID mappings in this mode.

In exclusive userns-remap mode, Sysbox ensures that all system
containers get **exclusive** user-ID and group-ID mappings on the
host. This has the benefit of hardening container-to-container
isolation (i.e., if a process escapes the container, it will find
itself without permissions to access any files).

For example, Sysbox may allocate user-ID mappings for a system container
as follows:

| Host User-IDs  | System Container User-IDs |
| -------------- | ------------------------- |
| x to (x+65535) | 0 to 65535                |

where 'x' is chosen from the range associated with user `sysbox` in
the host files `/etc/subuid` and `/etc/subgid`. These files are
automatically configured by Sysbox during installation (or more
specifically when the sysbox-mgr component is started during
installation). For example:

```console
# more /etc/subuid
sysbox:296608:268435456
```

The above means that Sysbox has reserved a range of 268435456
user-IDs on the host. This large range allows it to run up to 4K
system containers in parallel, each assigned a range of 64K
user-IDs. The reason 64K user-IDs are given to each system container
is to allow the container to have IDs ranging from the `root` (ID 0)
all the way up to user `nobody` (ID 65534).

If more than 4K containers are running at the same time, Sysbox will
by default re-use user-ID mappings from the range specified in
`/etc/subuid`. The same applies to group-ID mappings. In this scenario
multiple system containers may share the same user-ID mapping,
reducing container-to-container isolation a bit.

It is possible to configure Sysbox to not re-use mappings and
instead fail to launch the system container. But this requires restarting
Sysbox (in particular the sysbox-mgr component). See section
[Sysbox Reconfiguration](usage.md#sysbox-reconfiguration) for details on this.

Note that when Sysbox allocates exclusive user-ID mappings, the
presence of the [Ubuntu shiftfs module](#ubuntu-shiftfs-module) in the
kernel is required.

## Ubuntu Shiftfs Module

When Sysbox is configured in exclusive userns-remap mode (it's
default isolation mode), it makes use of the Ubuntu shiftfs module,
which is included in recent Ubuntu kernels (see the list of
[supported Linux distros](distro-compat.md) for more info on this).

The purpose of this module is to perform filesystem user-ID and
group-ID "shifting" between the a container's Linux user namespace and
the host's initial user namespace.

Recall from the [prior section](#exclusive-user-namespace-mappings)
that in Exclusive userns-remap mode, Sysbox allocates exclusive
user-namespace user-ID and group-ID mappings for each system
container.

Without the shiftfs module, the system container would see its root
filesystem files (as well as any mounted directories and files) owned
by `nobody:nogroup`, which in essence renders the container
unusable. The reason for this is that the container's rootfs is
typically owned by `root:root` on the host, but there is no mapping
from user `root` on the host to the exclusive user-ID range assigned
to the system container by Sysbox.

This problem is resolved by Sysbox's use of the Ubuntu shiftfs
module. By virtue of Sysbox mounting shiftfs on the system container's
rootfs as well as mount sources, the ownership of files will be mapped
as follows between the host and the system container:

| File Ownership on Host | File Ownership in System Container |
| ---------------------- | ---------------------------------- |
| 0 to 65535             | 0 to 65535                         |
| Others                 | nobody:nogroup                     |

This means that the system container processes will now see files with
the correct ownership (i.e., directories in the container's `/`
directory will have `root:root` ownership).

This however also means that files written by the system container's
root user will appear as `root:root` on the host (even though the root
user in the system container is mapped to a non-root, fully
unprivileged user on the host). Because of this, some security
precautions on the host are needed, as described in the [next section](#shiftfs-security-precautions).

To verify the Ubuntu shiftfs module is loaded, type:

```console
# lsmod | grep shiftfs
shiftfs           24576  0
```

The Ubuntu shiftfs module must be present in the kernel when
Sysbox is configured in [exclusive userns-remap mode](usage.md#exclusive-userns-remap-mode).

The Ubuntu shiftfs module is not used in [docker userns-remap mode](usage.md#docker-userns-remap-mode).

Sysbox will check for this. If the module is required but not present
in the Linux kernel, Sysbox will fail to launch containers and issue
an error such as [this one](troubleshoot.md#ubuntu-shiftfs-module-not-present).

### Shiftfs Security Precautions

When Sysbox uses shiftfs, some security precautions are recommended.

These arise from the fact that while the root user in the system
container is mapped to a non-root user on the host, files written by
the root user in the system container to mount-points under shiftfs are
mapped into `root:root` on the host. If the system container is
compromised or runs untrusted workloads, this can cause problems.

For example, an attacker running inside the system container can
create set-user-ID-root executables which can then be executed by
non-root users on the host to gain root privileges.

Note that this vulnerability is not specific to Nestybox system
containers or shiftfs; the same attack is possible with regular Docker
containers because in those the root user in the container is in fact
the root user on the host, so files written by the root user in the
container have `root:root` ownership on the host.

To reduce the attack surface, the following security precautions are
recommended:

-   The container's root filesystem should be in a directory accessible
    to the host's root user only (e.g., 0700 permissions).

    -   This is always the case when using Docker with Sysbox, because the
        Docker daemon makes `/var/lib/docker` accessible by the host's
        root user only.

-   The container's mount sources on the host should also be in a
    directory only accessible to the host's root user.

    -   This is always the case when using Docker volume and tmpfs mounts,
        since the mount source is also under `/var/lib/docker`.

    -   For bind mounts however this is not guaranteed because the user
        chooses the bind mount source. Thus, the user performing the bind
        mount should explicitly ensure this or take alternative precautions
        as described below.

For cases where the mount source (e.g., a bind mount source) is not in
a directory accessible by the root user only, an alternative
precaution is to mount the bind source as read-only inside the system
container. For example:

```console
$ docker run --runtime=sysbox-runc -it --mount type=bind,source=/path/to/bind/source,target=/path/to/mnt/point,readonly my-syscont
```

If this is not possible (e.g., because the system container must have
write access to the bind mount), another alternative is to explicitly
remount the mount source with the `noexec` attribute on the host prior
to starting the system container:

```console
$ sudo mount --bind /path/to/bind/source /path/to/bind/source
$ sudo mount -o remount,bind,noexec /path/to/bind/source /path/to/bind/source
$ docker run --runtime=sysbox-runc -it --mount type=bind,source=/path/to/bind/source,target=/path/to/mnt/point my-syscont`
```

Note that when a system container starts, Sysbox mounts shiftfs over
the rootfs and mount source. The shiftfs mount implicitly gives them a
`noexec` attribute on the host in order to protect against the attack
described above. However this only lasts while the associated system
container is running.

By explicitly remounting the bind source directory with the `noexec`
attribute as described above, a user can ensure that no user on the
host can execute files within the mount-source directory even after
the container is stopped.

### Shiftfs Functional Limitations

The Ubuntu shiftfs module is very recent and therefore has some
functional limitations as this time.

One such limitation is that overlayfs can't be mounted on top of shiftfs.

This implies that when a system container is using [exclusive userns-remap mode](usage.md#exclusive-userns-remap-mode),
applications running inside the system container that use overlayfs
mounts may not work properly.

Note that one such application is Docker, which mounts overlayfs over
portions of its `/var/lib/docker` directory. For this specific case
however, Sysbox sets up the system container such that the limitation
above is worked-around, allowing Docker to operate properly within the
system container.

## Procfs Virtualization

Sysbox performs partial virtualization of the system container's
procfs (i.e., `/proc`).

The main goals for this are:

1) Make the container feel more like a real host. This in turn
   increases the types of programs that can run inside the container.

2) Increase isolation between the container and the host.

Note also that by virtue of enabling the Linux user namespace in all
system containers, kernel resources under `/proc/sys` that are not
namespaced by Linux (e.g., `/proc/sys/kernel/*`) can't be changed from
within the system container. This prevents programs running inside the
system container from writing to these procfs files and affecting
system-wide settings.

## OCI compatibility

Sysbox is a fork of the [OCI runc](https://github.com/opencontainers/runc). It is mostly
(but not 100%) compatible with the OCI runtime specification.

The incompatibilities arise from Nestybox's desire to make deployment
of system containers possible with Docker.

We believe these incompatibilities won't negatively affect users of
Sysbox and should mostly be transparent to them.

Here is a list of OCI runtime incompatibilities:

### Namespaces

Sysbox requires that the system container's `config.json` file have a
namespace array field with at least the following namespaces:

-   pid
-   ipc
-   uts
-   mount
-   network

This is normally the case for Docker containers.

Sysbox adds the following namespaces to all system containers:

-   user
-   cgroup

### Process Capabilities

Sysbox always enables all process capabilities for the system
container's init process (which runs as the root user).

### Procfs

Sysbox always mounts `/proc/sys` read-write inside the
system container.

Note that by virtue of enabling the Linux user namespace, only
namespaced resources under `/proc/sys` will be writable from within
the system container. Non-namespaced resources (e.g., those under
`/proc/sys/kernel`) won't be writable from within the system container,
unless they are virtualized by Sysbox (see [Procfs Virtualization](#procfs-virtualization)).

### Cgroupfs Mount

Sysbox always mounts the cgroupfs as read-write inside the
system container (under `/sys/fs/cgroup`).

This allows programs inside the system container (e.g., Docker) to
assign cgroup resources to child containers. The assigned resources
are always a subset of the cgroup resources assigned to the system
container itself.

Sysbox ensures that programs inside the system container can't
modify the cgroup resources assigned to the container itself, or
cgroup resources associated with the rest of the system.

### Seccomp

Sysbox modifies the system container's seccomp configuration to
whitelist syscalls such as: mount, unmount, pivot_root, and a few
others.

This allows execution of system level programs within the system
container, such as Docker.

### AppArmor

Sysbox currently ignores the Docker AppArmor profile, as it's
too restrictive (e.g., prevents mounts inside the container,
prevents write access to `/proc/sys`, etc.)

### Read-only and Masked Paths

Sysbox honors read-only paths in the system container's
`config.json`, with the exception of paths at or under `/proc`
or under `/sys`.

The same applies to masked paths.

### Mounts

Sysbox honors the mounts specified in the system container's `config.json`
file, with a few exceptions such as:

-   Mounts into the system container's `/var/lib/docker` when Sysbox
    is configured in exclusive userns-remap mode (it's default
    operating mode).

-   Mounts into the system container's `/proc` and `/sys`.

In addition, Sysbox adds the following mounts to the system container:

-   Read-only bind mount of the host's `/lib/modules/<kernel-release>`
    into a corresponding path within the system container.

-   For system containers whose init process is Systemd, Sysbox mounts
    tmpfs inside the following directories in the system container:
    `/run`, `/run/lock`, `/tmp`.

-   Select mounts under the system container's `/sys` and `/proc`
    directories.

## Sysbox Nesting

Sysbox must run at the host level; it does not support running inside
a system container. This implies that we don't support running a
system container inside a system container at this time.

[cgroup-namespaces]: http://man7.org/linux/man-pages/man7/cgroup_namespaces.7.html
