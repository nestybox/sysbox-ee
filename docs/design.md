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

## User Namespace & ID Mappings

As mentioned in the prior section, Sysbox enables the Linux user
namespace in all system containers.

The user namespace works by mapping user-IDs and group-IDs between the
container and the host (or more precisely between the container's
user namespace and its parent user namespace). For example,
root in the container maps to a non-root user in the host.

When starting a system container, if the higher-layer (e.g., Docker +
containerd) provides these mappings to Sysbox via the container's
OCI `config.json` file, then Sysbox honors them. Otherwise, Sysbox
allocates these mappings for the system container. The mappings remain
allocated until the container is destroyed.

When allocating the mappings, Sysbox ensures all system containers
get *exclusive* user-ID and group-ID mappings on the host. This has
the benefit of hardening container-to-container isolation (i.e., if a
process escapes the container, it will find itself without permissions
to access files of the host and all other containers).

The allocated mappings come from the range specified in the host files
`/etc/subuid` and `/etc/subgid`. These files are automatically
configured by Sysbox during installation (more specifically when the
sysbox-mgr component is started during installation). For example:

```
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
of the [Nestybox shiftfs module](#nestybox-shiftfs-module) in the
kernel is required.

## Nestybox Shiftfs Module

Sysbox makes use of the Nestybox Shiftfs module, which can be
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
in conjunction with Sysbox.

The Nestybox shiftfs module is not upstreamed into the Linux
kernel. It's included in the Sysbox installer and loaded
automatically into the kernel when Sysbox is installed. Sysbox
users normally need not worry about it.

To verify the module is loaded, type:

```
# lsmod | grep shift
nbox_shiftfs           24576  0
```

**Notes:**

1) The Nestybox shiftfs module is required to be present in the kernel
when running Docker with Sysbox, specifically when Docker is
configured without userns-remap (the default and prefered
configuration, as described in the [Sysbox Usage Guide](usage.md#interaction-with-docker-userns-remap)).
If the module is not present in the Linux kernel in this case, Sysbox
will fail to launch containers and issue an appropriate error.

2) Ubuntu kernels starting with 5.0 (including the upcoming Ubuntu
19.10 release) already include a module called "shiftfs". While this
module is conceptually similar to Nestybox's shiftfs, it's **not** the
same. The Nestybox shiftfs module includes some changes that are
specific to Sysbox.

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

Sysbox must run at the host level; it does not support running
inside a system container. This implies that we don't support
running a system container inside a system container.
