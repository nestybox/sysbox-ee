Sysvisor Design Notes
=====================

This document briefly describes some aspects of Sysvisor's design.

## Sysvisor Components

Sysvisor is made up of the following components:

* sysvisor-runc

* sysvisor-fs

* sysvisor-mgr

sysvisor-runc is the program that creates system containers. It's the
"frontend" of sysvisor: higher layers (e.g., Docker) invoke
sysvisor-runc to launch system containers.

sysvisor-fs is a file-system-in-user-space (FUSE) that emulates
portions of the system container's filesystem, in particular the
`/proc/sys` directory. It's purpose is to make the system container
more closely resemble a real host while ensuring proper isolation.

sysvisor-mgr is a daemon that provides services to sysvisor-runc and
sysvisor-fs. For example, it manages assignment of exclusive user
namespace subuid mappings to system containers.

Together, sysvisor-fs and sysvisor-mgr are the "backends" for
sysvisor. Communication between the sysvisor components is done via
gRPC.

## Linux Namespace Usage

Sysvisor always enables all Linux namespaces in the system containers
(including the Linux user namespace).

This is done to improve isolation of the container from the rest of
the system (i.e., root inside the container maps to a fully
unprivileged user on the host).

This also allows the system container to run more types of workloads,
in particular system-level workloads that require root inside the
container to have full privileges within the container.

This is one area where Sysvisor deviates from the OCI specification,
which allows a higher layer (e.g., Docker + containerd) to choose the
namespaces that should be enabled for the container.

## User Namespace & ID Mappings

As mentioned in the prior section, Sysvisor enables the Linux user
namespace in all system containers.

The user namespace works by mapping user-IDs and group-IDs between the
container and the host (or more precisely between the container's
user namespace and its parent user namespace). For example,
root in the container maps to a non-root user in the host.

When starting a system container, if the higher-layer (e.g., Docker +
containerd) provides these mappings to Sysvisor via the container's
OCI `config.json` file, then Sysvisor honors them. Otherwise, Sysvisor
allocates these mappings for the system container. The mappings remain
allocated until the container is destroyed.

When allocating the mappings, Sysvisor ensures all system containers
get *exclusive* user-ID and group-ID mappings on the host. This has
the benefit of hardening container-to-container isolation (i.e., if a
process escapes the container, it will find itself without permissions
to access files of the host and all other containers).

The allocated mappings come from the range specified in the host files
`/etc/subuid` and `/etc/subgid`. These files are automatically
configured by Sysvisor during installation (more specifically when the
sysvisor-mgr component is started during installation). For example:

```
# more /etc/subuid
sysvisor:296608:268435456
```

The above means that Sysvisor has reserved a range of 268435456
user-IDs on the host. This large range allows it to run up to 4K
system containers in parallel, each assigned a range of 64K
user-IDs. The reason 64K user-IDs are given to each system container
is to allow the container to have IDs ranging from the `root` (ID 0)
all the way up to user `nobody` (ID 65534).

If more than 4K containers are running at the same time, Sysvisor will
by default re-use user-ID mappings from the range specified in
`/etc/subuid`. The same applies to group-ID mappings. In this scenario
multiple system containers may share the same user-ID mapping,
reducing container-to-container isolation.

It is possible to configure Sysvisor to not re-use mappings and
instead fail to launch the container. But this requires restarting
Sysvisor (in particular the sysvisor-mgr component). See section
[Sysvisor Reconfiguration](#sysvisor-reconfiguration) for details on this.

One final note: when Sysvisor allocates user-ID mappings, the presence
of the [Nestybox shiftfs module](#nestybox-shiftfs-module) in the
kernel is required.

## Nestybox Shiftfs Module

Sysvisor makes use of the Nestybox Shiftfs module, which can be
found [here](https://github.com/nestybox/nbox-shiftfs-external).

The purpose of this module is to perform user-ID and group-ID
"shifting" between the a container's Linux user namespace and the
host's initial user namespace.

This functionality is needed in order to allow a system container
whose root user is mapped to a non-root user in the host (for extra
isolation), to access the container's root filesystem on the host
(which is normally owned by root:root).

The Nestybox shiftfs module is based on a similar module originally
written by James Bottomley, but has been modified by Nestybox for use
in conjunction with Sysvisor.

The Nestybox shiftfs module is not upstreamed into the Linux
kernel. It's included in the Sysvisor installer and loaded
automatically into the kernel when Sysvisor is installed. Sysvisor
users normally need not worry about it.

To verify the module is loaded, type:

```
# lsmod | grep shift
nbox_shiftfs           24576  0
```

**Notes:**

1) The Nestybox shiftfs module is required to be present in the kernel
when running Docker with Sysvisor, specifically when Docker is
configured without userns-remap (the default and prefered
configuration, as described in the [Sysvisor Usage Guide](usage.md#interaction-with-docker-userns-remap)).
If the module is not present in the Linux kernel in this case, Sysvisor
will fail to launch containers and issue an appropriate error.

2) Ubuntu kernels starting with 5.0 (including the upcoming Ubuntu
19.10 release) already include a module called "shiftfs". While this
module is conceptually similar to Nestybox's shiftfs, it's **not** the
same. The Nestybox shiftfs module includes some changes that are
specific to Sysvisor.

## Procfs Virtualization

Sysvisor performs partial virtualization of the system container's
procfs (i.e., `/proc`).

The main goals for this are:

1) Make the container feel more like a real host. This in turn
   increases the types of programs that can run inside the container.

2) Increase isolation between the container and the host.

Currently, Sysvisor does virtualization of the following procfs resources:

* `/proc/uptime`

  - Shows the uptime of the system container, not the host.

* `/proc/sys/net/netfilter/nf_conntrack_max`

  - Sysvisor emulates this resource independently per system
    container, and sets appropriate values in the host kernel's
    `nf_conntrack_max`.


Note also that by virtue of enabling the Linux user namespace in all
system containers, kernel resources under `/proc/sys` that are not
namespaced by Linux (e.g., `/proc/sys/kernel/*`) can't be changed from
within the system container. This prevents programs running inside the
system container from writing to these procfs files and affecting
system-wide settings.

## OCI compatibility

Sysvisor is mostly (but not 100%) compatible with the [OCI runtime spec](https://github.com/opencontainers/runtime-spec).

The incompatibilities arise from Nestybox's desire to make deployment
of system containers possible with Docker.

We believe these incompatibilities won't negatively affect users of
Sysvisor and should mostly be transparent to them.

Here is a list of OCI runtime incompatibilities:

### Namespaces

Sysvisor requires that the system container's `config.json` file have a
namespace array field with at least the following namespaces:

* pid
* ipc
* uts
* mount
* network

This is normally the case for Docker containers.

Sysvisor adds the following namespaces to all system containers:

* user
* cgroup

### Process Capabilities

Sysvisor always enables all process capabilities for the system
container's init process (which runs as the root user).

### Procfs

Sysvisor always mounts `/proc/sys` read-write inside the
system container.

Note that by virtue of enabling the Linux user namespace, only
namespaced resources under `/proc/sys` will be writeable from within
the system container. Non-namespaced resources (e.g., those under
`/proc/sys/kernel`) won't be writeable from within the system container,
unless they are virtualized by Sysvisor (see [Procfs Virtualization](#procfs-virtualization)).

### Cgroupfs Mount

Sysvisor always mounts the cgroupfs as read-write inside the
system container (under `/sys/fs/cgroup`).

This allows programs inside the system container (e.g., Docker) to
assign cgroup resources to child containers. The assigned resources
are always a subset of the cgroup resources assigned to the system
container itself.

Sysvisor ensures that programs inside the system container can't
modify the cgroup resources assigned to the container itself, or
cgroup resources associated with the rest of the system.

### Seccomp

Sysvisor modifies the system container's seccomp configuration to
whitelist syscalls such as: mount, unmount, pivot_root, and a few
others.

This allows execution of system level programs within the system
container, such as Docker.

### AppArmor

Sysvisor currently ignores the Docker AppArmor profile, as it's
too restrictive (e.g., prevents mounts inside the container,
prevents write access to `/proc/sys`, etc.)

### Read-only Paths

Sysvisor honors read-only paths in the system container's
`config.json`, with the exception of `/proc`.

### Masked paths

Sysvisor honors masked paths in the system container's `config.json`,
with the exception of `/proc`.

### Mounts

Sysvisor honors the mounts specified in the system container's `config.json`
file. However, it adds the following mounts to the system container:

* Read-only bind mount of the host's `/lib/modules/<kernel-release>`
  into a corresponding path within the system container.

## Sysvisor Nesting

Sysvisor must run at the host level; it does not support running
inside a system container. This implies that we don't support
running a system container inside a system container.

## Sysvisor Reconfiguration

TODO: explain how to re-start sysvisor
