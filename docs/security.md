# Sysbox Security Guide

This document describes security aspects of Sysbox system containers.

**Note:** it's early days for Nestybox and while our system containers
already incorporate important security features such use of the Linux user
namespace and exclusive user-ID mappings per container, other security
aspects need further development. The text below describes what
security features are currently in place and where more work is needed.

## Contents

-   [System Container Security](#system-container-security)
    -   [Root Filesystem Jail](#root-filesystem-jail)
    -   [Linux Namespaces](#linux-namespaces)
    -   [Exclusive User Namespace Mappings](#exclusive-user-namespace-mappings)
    -   [Procfs](#procfs)
    -   [Sysfs](#sysfs)
    -   [Process Capabilities](#process-capabilities)
    -   [System Calls](#system-calls)
    -   [AppArmor](#apparmor)
    -   [Devices](#devices)
    -   [Resource Limiting & Cgroups](#resource-limiting--cgroups)
    -   [Host PID or Network Sharing](#host-pid-or-network-sharing)
    -   [Privileged Container Support](#privileged-container-support)

## System Container Security

### Root Filesystem Jail

System container processes are confined to the directory hierarchy
associated with the container's root filesystem, plus any
configured mounts (e.g., Docker volumes or bind-mounts).

### Linux Namespaces

System containers always use **all** Linux namespaces, including the
user-namespace, as described [here](design.md#linux-namespace-usage).

This provides strong isolation from the underlying host and from other
containers.

### Exclusive User Namespace Mappings

By default, system containers use exclusive user-namespace user-ID and
group-ID mappings per container. This enhances container-to-host and
container-to-container isolation.

See [here](design.md#exclusive-user-namespace-mappings) for further
details.

### Procfs

The system container's procfs (i.e., `/proc`) is mounted read-write,
but protected by the Linux user-namespace which ensures that only
resources assigned to the system container are accessible via `/proc`.

Having said that, there are several improvements that Sysbox needs in
this area to ensure information about system resources outside of the
system container is not exposed inside the container, ensuring
unmounts/re-mounts work, etc.

### Sysfs

The system container's sysfs (i.e., `/sys`) is mounted read-only,
with the exception of `/sys/fs/cgroup`.

This ensures system container processes can't modify system-level
controls exposed via `/sys`.

Having said that, this is also an area where improvements are needed
in Sysbox to ensure sensitive information is not exposed,
unmounts/re-mounts work correctly, etc.

The `/sys/fs/cgroup` directory is mounted read-write to allow system
container processes to assign cgroup resources. Sysbox sets up the
system container in such a way that processes inside the system
container can't modify cgroup resources assigned to the system
container itself. In addition, system container processes can only use
cgroups to assign a subset of the system container resources.

### Process Capabilities

A system container's init process configured with user-ID 0 (root)
always starts with all capabilities enabled.

    $ docker run --runtime=sysbox-runc -it alpine:latest
    / # grep -i cap /proc/self/status
    CapInh: 0000003fffffffff
    CapPrm: 0000003fffffffff
    CapEff: 0000003fffffffff
    CapBnd: 0000003fffffffff
    CapAmb: 0000003fffffffff

Note that the system container's Linux user-namespace ensures that
these capabilities are only applicable to resources assigned to the
system container itself. It does not mean that the process has full
capabilities on the host (in fact is has no capabilities on resources
not assigned to the system container).

A system container's init process configured with a non-zero user-ID
starts with the capabilities passed to Sysbox by the container engine.

For example, when deploying system containers with Docker:

    $ docker run --runtime=sysbox-runc --user 1000 -it alpine:latest
    / $ grep -i cap /proc/self/status
    CapInh: 00000000a80425fb
    CapPrm: 0000000000000000
    CapEff: 0000000000000000
    CapBnd: 00000000a80425fb
    CapAmb: 0000000000000000

### System Calls

Nestybox system containers allow a minimum set of 300+ syscalls, using
Linux seccomp.

Significant syscalls blocked within system containers are the
same as those listed [in this Docker article](https://docs.docker.com/engine/security/seccomp/),
except that system container allow these system calls too:

    mount
    umount
    umount2
    add_key
    request_key
    keyctl
    pivot_root
    gethostname
    sethostname
    setns
    unshare

It's currently not possible to reduce the set of syscalls allowed within
a system container (i.e., the Docker `--security-opt seccomp=<profile>` option
is not supported).

### AppArmor

Nestybox system containers do not yet support AppArmor for mandatory
access control.

When using Docker to deploy a system container, the [default AppArmor profile](https://docs.docker.com/engine/security/apparmor/)
used by Docker is ignored as it's too restrictive for system containers
(i.e., the Docker `--security-opt apparmor=<profile>` option is not supported).

In the near future we plan to develop a default profile for system
containers.

### Devices

The following devices are always present in the system container:

    /dev/null
    /dev/zero
    /dev/full
    /dev/random
    /dev/urandom
    /dev/tty

Additional devices may be added by the container engine. For example,
when deploying system containers with Docker, you typically see the
following devices in addition to the ones listed above:

    /dev/console
    /dev/pts
    /dev/mqueue
    /dev/shm

Sysbox does not currently support exposing host devices inside system
containers (e.g., via the `docker run --device` option). We are
working on adding support for this.

### Resource Limiting & Cgroups

System container resource consumption can be limited via cgroups.

This can be used to balance resource consumption as well as to prevent
denial-of-service attacks in which a buggy or compromised system
container consumes all available resources in the system.

For example, when using Docker to deploy system containers, the
`docker run --cpu*`, `--memory*`, `--blkio*`, etc., settings can be
used for this purpose.

### Host PID or Network Sharing

System containers do not support sharing the pid or network namespaces
with the host (as this is not secure and it's incompatible with the
system container's user namespace).

For example, when using Docker to launch system containers, the
`docker run --pid=host` and `docker run --network=host` options
do not work with system containers.

### Privileged Container Support

System containers are incompatible with the Docker `--privileged`
flag.  See the [usage guide](usage.md#privileged-container-support)
for info on this.
