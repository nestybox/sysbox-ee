Sysbox Design Notes
===================

This document briefly describes some aspects of Sysbox's design.

## Sysbox Components

Sysbox is made up of the following components:

* sysbox-runc

* sysbox-fs

* sysbox-mgr

sysbox-runc is a container runtime, the program that does the low
level kernel setup for execution of system containers. It's the
"frontend" of sysbox: higher layers (e.g., Docker & containerd) invoke
sysbox-runc to launch system containers.

sysbox-fs is a file-system-in-user-space (FUSE) daemon that emulates
portions of the system container's filesystem, in particular portions
of the `/proc` directory. It's purpose is to make the system container
more closely resemble a real host while ensuring proper isolation from
the host.

sysbox-mgr is a daemon that provides services to sysbox-runc and
sysbox-fs. For example, it manages assignment of exclusive user
namespace user-ID and group-ID mappings to system containers.

Together, sysbox-fs and sysbox-mgr are the "backends" for
sysbox. Communication between the sysbox components is done via
gRPC.

## Linux Namespace Usage

Sysbox always enables all Linux namespaces in the system containers
(including the Linux user namespace).

This is done to improve isolation of the container from the rest of
the system (i.e., root inside the container maps to a fully
unprivileged user on the host).

This also allows the system container to run more types of workloads,
in particular system-level workloads that require root inside the
container to have full privileges within the container.

This is one area where Sysbox deviates from the OCI specification,
which allows a higher layer (e.g., Docker + containerd) to choose the
namespaces that should be enabled for the container.

## User Namespace and ID Mappings

As mentioned in the prior section, Sysbox enables the Linux user
namespace in all system containers.

The user namespace works by mapping user-IDs and group-IDs between the
container and the host (or more precisely between the container's
user namespace and its parent user namespace). For example,
root in the container maps to a non-root user in the host.

When starting a system container, if the higher-layer (e.g., Docker +
containerd) provides these mappings to Sysbox via the container's
OCI `config.json` file, then Sysbox honors them. Docker does this
when the Docker daemon is configured with the userns-remap option.

Otherwise (e.g., when Docker is configured without userns-remap),
Sysbox allocates these mappings for the system container. The mappings
remain allocated until the container is destroyed.

For example, Sysbox may allocate user-ID mappings for a system container
as follows:

  | User-ID Range on Host | User-ID range in System Container  |
  |-----------------------|------------------------------------|
  | X -> X + 65535        | 0 (root) -> 65535                  |

where X is chosen by Sysbox from the corresponding range in
`/etc/subuid` as described below. The same mapping applies to Group ID
ranges.

When allocating the mappings, Sysbox ensures all system containers
get *exclusive* user-ID and group-ID mappings on the host. This has
the benefit of hardening container-to-container isolation (i.e., if a
process escapes the container, it will find itself without permissions
to access files of the host and all other containers).

The allocated mappings come from the range specified in the host files
`/etc/subuid` and `/etc/subgid`. These files are automatically
configured by Sysbox during installation (more specifically when the
sysbox-mgr component is started during installation). For example:

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
reducing container-to-container isolation.

It is possible to configure Sysbox to not re-use mappings and
instead fail to launch the container. But this requires restarting
Sysbox (in particular the sysbox-mgr component). See section
[Sysbox Reconfiguration](usage.md#sysbox-reconfiguration) for details on this.

One final note: when Sysbox allocates user-ID mappings, the presence
of the Ubuntu shiftfs module in the kernel is required as described
below.

## Ubuntu Shiftfs Module

Sysbox makes use of the Ubuntu shiftfs module, which is included
in recent Ubuntu kernels (see the list of [supported Linux distros](../README.md#supported-linux-distros)
for more info on this).

The purpose of this module is to perform filesystem user-ID and
group-ID "shifting" between the a container's Linux user namespace and
the host's initial user namespace.

Recall from the [prior section](#user-namespace-and-id-mappings)
that Sysbox uses the Linux user namespace and allocates an exclusive
user-ID range on the host for each system container.

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
  |------------------------|------------------------------------|
  | 0 (root) -> 65535      | 0 (root) -> 65535                  |
  | Others                 | nobody:nogroup                     |

This means that the system container processes will now see files with
the correct ownership (i.e., directories in the container's `/`
directory will have `root:root` ownership).

This however also means that files written by the system container's
root user will appear as `root:root` on the host (even though the root
user in the system container is mapped to a non-root, fully
unprivileged user on the host). Because of this, some security
precautions on the host are needed, as described in the next section.

To verify the Ubuntu shiftfs module is loaded, type:

```console
# lsmod | grep shiftfs
shiftfs           24576  0
```

The Ubuntu shiftfs module is required to be present in the kernel when
running Docker with Sysbox, specifically when Docker is configured
without userns-remap (the default and prefered configuration, as
described in the [Sysbox Usage Guide](usage.md#interaction-with-docker-userns-remap)).
If Docker is configured with userns-remap enabled, the Ubuntu shiftfs module
is not required.

Sysbox will check for this. If the module is required but not present
in the Linux kernel, Sysbox will fail to launch containers and issue
an [appropriate error](troubleshoot.md#ubuntu-shiftfs-module-not-present).

### Shiftfs Security Precautions

When Sysbox uses shiftfs, some security precautions are recommended.

These arise from the fact that while the root user in the system
container is mapped to a non-root user on the host, files written by
the root user in the system container to mountpoints under shiftfs are
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

* The container's root filesystem should be in a directory accessible
  to the host's root user only (e.g., 0700 permissions).

  - This is always the case when using Docker with Sysbox, because the
    Docker daemon makes `/var/lib/docker` accessible by the host's
    root user only.

* The container's mount sources on the host should also be in a
  directory only accessible to the host's root user.

  - This is always the case when using Docker volume and tmpfs mounts,
    since the mount source is also under `/var/lib/docker`.

  - For bind mounts however this is not guaranteed because the user
    chooses the bind mount source. Thus, the user performing the bind
    mount should explictly ensure this or take alternative precautions
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

## Procfs Virtualization

Sysbox performs partial virtualization of the system container's
procfs (i.e., `/proc`).

The main goals for this are:

1) Make the container feel more like a real host. This in turn
   increases the types of programs that can run inside the container.

2) Increase isolation between the container and the host.

Currently, Sysbox does virtualization of the following procfs resources:

* `/proc/uptime`

  - Shows the uptime of the system container, not the host.

* `/proc/sys/net/netfilter/nf_conntrack_max`

  - Sysbox emulates this resource independently per system
    container, and sets appropriate values in the host kernel's
    `nf_conntrack_max`.


Note also that by virtue of enabling the Linux user namespace in all
system containers, kernel resources under `/proc/sys` that are not
namespaced by Linux (e.g., `/proc/sys/kernel/*`) can't be changed from
within the system container. This prevents programs running inside the
system container from writing to these procfs files and affecting
system-wide settings.

## OCI compatibility

Sysbox is mostly (but not 100%) compatible with the [OCI runtime spec](https://github.com/opencontainers/runtime-spec).

The incompatibilities arise from Nestybox's desire to make deployment
of system containers possible with Docker.

We believe these incompatibilities won't negatively affect users of
Sysbox and should mostly be transparent to them.

Here is a list of OCI runtime incompatibilities:

### Namespaces

Sysbox requires that the system container's `config.json` file have a
namespace array field with at least the following namespaces:

* pid
* ipc
* uts
* mount
* network

This is normally the case for Docker containers.

Sysbox adds the following namespaces to all system containers:

* user
* cgroup

### Process Capabilities

Sysbox always enables all process capabilities for the system
container's init process (which runs as the root user).

### Procfs

Sysbox always mounts `/proc/sys` read-write inside the
system container.

Note that by virtue of enabling the Linux user namespace, only
namespaced resources under `/proc/sys` will be writeable from within
the system container. Non-namespaced resources (e.g., those under
`/proc/sys/kernel`) won't be writeable from within the system container,
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

### Read-only Paths

Sysbox honors read-only paths in the system container's
`config.json`, with the exception of `/proc`.

### Masked paths

Sysbox honors masked paths in the system container's `config.json`,
with the exception of `/proc`.

### Mounts

Sysbox honors the mounts specified in the system container's `config.json`
file. However, it adds the following mounts to the system container:

* Read-only bind mount of the host's `/lib/modules/<kernel-release>`
  into a corresponding path within the system container.

## Sysbox Nesting

Sysbox must run at the host level; it does not support running inside
a system container. This implies that we don't support running a
system container inside a system container at this time.
