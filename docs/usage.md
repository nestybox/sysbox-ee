Sysvisor User's Guide
=====================

The Sysvisor [README](../README.md) file contains the basic information on how to install and
run Sysvisor. This document supplements the README file with additional information.

## Running containers with Sysvisor

We currently support two ways of running containers with Sysvisor:

1) Using Docker

2) Using the sysvisor-runc command directly.

### Using Docker

To launch a container with Docker + Sysvisor, simply use the `--runtime` flag in
the `docker run` command:

```bash
$ docker run --runtime=sysvisor-runc --rm -it --hostname my_cont debian:latest
root@my_cont:/#
```

If you wish to configure Sysvisor as the default runtime for Docker, refer to the
[Docker website](https://docs.docker.com/engine/reference/commandline/dockerd/).

### Using the sysvisor-runc command

It's easiest to use a higher-level container manager to launch containers
(e.g., Docker), but it's also possible to launch containers directly
via the sysvisor-runc command line.

As the root user, follow these steps:

1) Create a rootfs image for the container:

```bash
# mkdir /root/mycontainer
# cd /root/mycontainer
# mkdir rootfs
# docker export $(docker create debian:latest) | tar -C rootfs -xvf -
```

2) Create the OCI spec for the container, using the `sysvisor-runc --spec` command:

```
# sysvisor-runc spec
```

This will create a default OCI spec (i.e., `config.json` file) for the container.

3) Launch the container

Choose a unique name for the container and run:

```
# sysvisor-runc run mycontainerid
```

Use `sysvisor-runc --help` command for help on all commands supported by Sysvisor.

Also, in step (2) above, feel free to modify the container's
`config.json` to your needs. But note that Sysvisor ignores a few
of the OCI directives in this file (refer to the [Sysvisor design document](design.md)
for details).

### Other

We officially only support the above methods to run Sysvisor.

However, because Sysvisor is almost 100% OCI compatible, we plan to
add support for other OCI compatible container managers (e.g.,
[cri-o](https://cri-o.io/)) soon.

## Rootless Container Support

Sysvisor must run as root on the host system. It won't work
if executed without root privileges.

## Interaction with Docker Userns Remap

Docker has a configuration called [userns-remap](https://docs.docker.com/engine/security/userns-remap)
that enables the Linux user namespace in containers. By default, userns-remap is disabled in Docker.

Sysvisor works out-of-the-box with either configuration of Docker
(with or without userns-remap enabled). No change to Sysvisor's
configuration is needed either way.

However, Sysvisor works best with Docker userns-remap *disabled*
(though this mode is supported on a reduced number of distros,
as described [here](../README.md#supported-linux-distros)).

The reason is that when userns-remap is disabled, Sysvisor will
allocate exclusive user-ID and group-ID mappings for the container's
user namespace, thereby improving container-to-container isolation.

If on the other hand Docker userns-remap is enabled, then Docker
chooses the user-ID and group-ID mappings for the container;
Sysvisor honors these. However Docker currently has a limitation: it
uses the same user-ID and group-ID mappings for all containers,
thereby decreasing isolation between containers (i.e., if a process
escapes the container, it may be able to access the filesystem
of other containers).

The table below summarizes this:

| Docker userns-remap | Description |
|---------------------|-------------|
| disabled            | sysvisor will allocate exclusive uid(gid) mappings per sys container and perform uid(gid) shifting. |
|                     | Strong container-to-host isolation. |
|                     | Strong container-to-container isolation. |
|                     | Uses the Nestybox Shiftfs module in the kernel. |
|                     |
| enabled             | sysvisor will honor Docker's uid(gid) mappings. |
|                     | Strong container-to-host isolation. |
|                     | Reduced container-to-container isolation (same uid(gid) range). |
|                     | Does not use the Nestybox Shiftfs module in the kernel. |

For further info Sysvisor's usage of the Linux user namespace and
associated ID mappings, refer to the [Sysvisor design document](design.md).

## Docker Bind Mount Permissions

Sysvisor supports all Docker storage mount types: volume, bind, or tmpfs.

However, for bind mounts there are some caveats. These caveats do not apply to
volume mounts and tmpfs mounts.

The caveats for bind mounts arise from the fact that Sysvisor containers
use the Linux user namespace, and therefore the root user inside the
container maps to a non-root user in the host.

The caveats vary depending on whether Docker is configured with
userns-remap or not, as explained below.

### Without Docker userns-remap

This is the default Docker configuration. In this case, there are two
rules that apply:

1) The bind mount source in the host must be owned by `root:root`.

2) The path to the bind mount source in the host must be
   accessible *only* by the root user in the host.

For example, if running the container with:

```bash
$ docker run --runtime=sysvisor-runc -it --mount type=bind,source=/a/b/c,target=/mnt/somedir debian:latest
```

the bind source path at `/a/b/c` must be accesible by root only. This
means that at least one of `/a` or `/a/b` or `/a/b/c` must have `0700`
permissions on the host.

Note that if either `/a` or `/a/b` have root ownership with `0700` permissions,
then the last directory in the path (`c`) must be owned by `root:root` but could
have less restrictive permissions (e.g., `0644` or even `0777`).

The rationale for these rules is the following.

Rule (1) ensures that the root user in the container will have correct
permissions to access the bind mount. Sysvisor uses the Nestybox
shiftfs module to do some magic here.

Rule (2) is a security precaution. It reduces the chances of a
malicious program inside the container from compromising security in
the host system.

### With Docker userns-remap

If Docker is configured with userns-remap, then the following
rule applies to bind mounts:

The bind mount source in the host must be owned by `uid:gid`, where
`uid:gid` are the host's user-ID and group-ID associated with the
docker userns-remap configuration.

For example, if the docker userns-remap configuration is the following:

```bash
$ cat /etc/docker/daemon.json
{
   "userns-remap": "someuser"
}
```

then the bind mount source must be owned by `someuser:someuser`.

## Unsupported Docker Features

TODO ...

E.g., what docker volume drivers don't work with shiftfs


## Container Environment Checks

TODO: show how to check that uid are assigned, userns is working, shiftfs is on, etc.