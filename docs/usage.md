Sysvisor User's Guide
=====================

The Sysvisor [README](../README.md) file contains the basic
information on how to install Sysvisor and create containers with
it. This document supplements the README file with additional
information.

## Running System Containers with Sysvisor

We currently support two ways of running system containers with Sysvisor:

1) Using Docker

2) Using the sysvisor-runc command directly

### Using Docker

It's easy to run system container using Docker. Simply use the `--runtime`
flag in the `docker run` command:

```bash
$ docker run --runtime=sysvisor-runc --rm -it --hostname my_cont debian:latest
root@my_cont:/#
```

It's possible to configure Sysvisor as the default runtime for Docker. This
way you don't have to use the `--runtime` flag everytime. If you wish to do this,
refer to the [Docker website](https://docs.docker.com/engine/reference/commandline/dockerd/).

### Using the sysvisor-runc command

It's also possible to launch system containers directly via the
sysvisor-runc command line.

As the root user, follow these steps:

1) Create a rootfs image for the system container:

```bash
# mkdir /root/syscontainer
# cd /root/syscontainer
# mkdir rootfs
# docker export $(docker create debian:latest) | tar -C rootfs -xvf -
```

2) Create the OCI spec for the system container, using the
`sysvisor-runc --spec` command:

```
# sysvisor-runc spec
```

This will create an OCI `config.json` file.

3) Launch the system container

Choose a unique name for the container and run:

```
# sysvisor-runc run my_syscontainer
```

Use `sysvisor-runc --help` command for help on all commands supported by Sysvisor.

Also, in step (2) above, feel free to modify the system container's
`config.json` to your needs. But note that Sysvisor ignores a few
of the OCI directives in this file (refer to the [Sysvisor design document](design.md#oci-compatibility)
for details).

### Other

We officially only support the above methods to run Sysvisor.

However, because Sysvisor is almost 100% OCI compatible, we plan to
add support for other OCI compatible container managers (e.g.,
[cri-o](https://cri-o.io/)) soon.

## Sysvisor execution & configuration

Sysvisor's [daemons](design.md#sysvisor-components) execution and admin-state should be managed through systemd's cli interface.

```bash
$ sudo systemctl start sysvisor
$ sudo systemctl stop sysvisor
$ sudo sysctl restart sysvisor
```

These commands are particularly useful in scenarios where Sysvisor's daemons need to be initialized with customized parameters (e.g. --log-level debug). In these cases, user will be expected to proceed as below:

1) Modify the desired service initialization instruction.

   Example:

   ```bash
   $ sudo sed -i '/^ExecStart/ s/$/ --log-level debug/' /etc/systemd/system/sysvisor.service.wants/sysvisor-fs.service
   $
   $ egrep "ExecStart" /etc/systemd/system/sysvisor.service.wants/sysvisor-fs.service
   ExecStart=/usr/local/sbin/sysvisor-fs --log /var/log/sysvisor-fs.log --log-level debug
   ```

2) Reload **systemd** to digest the previous change:

   ```bash
   $ sudo systemctl daemon-reload
   ```

3) Restart **sysvisor** service:

   ```bash
   $ sudo systemctl restart sysvisor
   ```

Finally, bear in mind that even though Sysvisor software is comprised of various daemons and its respective services, you should only interact with its outer-most systemd service: **sysvisor**.


## Rootless Container Support

Sysvisor must run as root on the host system. It won't work
if executed without root privileges.

## Interaction with Docker Userns Remap

Docker has a configuration called [userns-remap](https://docs.docker.com/engine/security/userns-remap)
that enables the Linux user namespace in containers. By default, userns-remap is disabled in Docker.

Sysvisor works out-of-the-box with either configuration of Docker
(i.e., with or without userns-remap enabled). No change to Sysvisor's
configuration is needed either way.

However, Sysvisor works best with Docker userns-remap *disabled*
(though this is supported by Sysvisor in a reduced number of distros,
as described [here](../README.md#supported-linux-distros)).

The reason userns-remap disabled is preferred, is that in this case
Sysvisor will allocate exclusive user-ID and group-ID mappings for the
system container's user namespace, thereby improving
container-to-container isolation.

If on the other hand Docker userns-remap is enabled, then Docker
chooses the user-ID and group-ID mappings for the container and
Sysvisor honors these. However Docker currently has a limitation: it
uses the same user-ID and group-ID mappings for all containers,
thereby decreasing isolation between containers (i.e., if a process
escapes the container, it may be able to access the filesystem
of other containers).

The table below summarizes this:

| Docker userns-remap | Description |
|---------------------|-------------|
| disabled            | sysvisor will allocate exclusive uid(gid) mappings per system container and perform uid(gid) shifting. |
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

Sysvisor system containers support all Docker storage mount types:
volume, bind, or tmpfs.

However, for bind mounts there are some caveats. These caveats do not apply to
volume mounts and tmpfs mounts.

The caveats for bind mounts arise from the fact that system containers
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

For example, if running the system container with:

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

Rule (1) ensures that the root user in the system container will have
correct permissions to access the bind mount. Sysvisor uses the
Nestybox shiftfs module to do some magic here.

Rule (2) is a security precaution. It reduces the chances of a
malicious program inside the system container from compromising
security in the host system.

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


## System Container Environment Checks

TODO: show how to check that uid are assigned, userns is working, shiftfs is on, etc.
