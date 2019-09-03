Sysboxd User's Guide
====================

The Sysboxd [README](../README.md) file contains the basic information
on how to install Sysboxd and create system containers with it. This
document supplements the README file with additional information.

## Running System Containers with Sysboxd

We currently support two ways of running system containers with Sysboxd:

1) Using Docker (the easy and preferred way)

2) Using the `sysbox-runc` command directly (more control but harder)

### Using Docker

It's easy to run system container using Docker. Simply use the `--runtime=sysbox-runc`
flag in the `docker run` command:

```bash
$ docker run --runtime=sysbox-runc --rm -it --hostname my_cont debian:latest
root@my_cont:/#
```

It's possible to configure Sysboxd as the default runtime for Docker. This
way you don't have to use the `--runtime` flag everytime. If you wish to do this,
refer to the [Docker website](https://docs.docker.com/engine/reference/commandline/dockerd/).

### Using the sysbox-runc command

It's also possible to launch system containers directly via the
`sysbox-runc` command.

As the root user, follow these steps:

1) Create a rootfs image for the system container:

```bash
# mkdir /root/syscontainer
# cd /root/syscontainer
# mkdir rootfs
# docker export $(docker create debian:latest) | tar -C rootfs -xvf -
```

2) Create the OCI spec (i.e., `config.json` file) for the system container:

```
# sysbox-runc spec
```

3) Launch the system container.

Choose a unique name for the container and run:

```
# sysbox-runc run my_syscontainer
```

Use `sysbox-runc --help` command for help on all commands supported.

Also, in step (2) above, feel free to modify the system container's
`config.json` to your needs. But note that Sysboxd ignores a few
of the OCI directives in this file (refer to the [Sysboxd design document](design.md#oci-compatibility)
for details).

### Support for Other Container Managers

We officially only support the above methods to run Sysboxd.

However, because Sysboxd is almost 100% OCI compatible, we plan to
add support for other OCI compatible container managers (e.g.,
[cri-o](https://cri-o.io/)) soon.

## Running Software inside the System Container

A system container is logically a super-set of a regular Docker
application container, and thus should be able to run any application
that runs in a regular Docker container, plus system-level software
(e.g., Docker inside the system container).

Nestybox's goal is to allow you run any software inside the system
container just as you would on a physical host. Ideally there
shouldn't be any difference.

The sub-sections below provide information on running system-level
software inside a system container (i.e., software that normally does
not run inside a regular Docker container).

### Docker-in-Docker

Nestybox system containers support running Docker inside the system
container, without using privileged containers or bind-mounting the
host's Docker socket into the container. In other words, cleanly and
securely, with total isolation between the inner and outer Docker
daemons.

Moreover, it's fast: the Docker daemon inside the container uses the
fast overlay2 (or btrfs) storage drivers, rather than alternative
docker-in-docker solutions that resort to the slower vfs driver.

To run Docker inside a system container (a.k.a Docker-in-Docker), the
easiest way is to use a system container image that has Docker
pre-installed in it.

You can find a few such images in the Nestybox DockerHub repo:

```
https://hub.docker.com/r/nestybox
```

For example, to run a system container that has Ubuntu Disco + Docker inside, simply
type:

```bash
$ docker run --runtime=sysbox-runc -it --hostname sc nestybox/ubuntu-disco-docker:latest
root@sc:/#
```

From within the system container, start Docker:

```bash
root@sc:/# dockerd > /var/log/dockerd.log 2>&1 &
```

Note that we don't yet support systemd inside the system container, so
the Docker daemon needs to be started manually as shown above.

The Docker daemon should now be running inside the system container.
You can verify this by looking at the Docker daemon log file:

```bash
root@sc:/# tail -n 2 /var/log/dockerd.log
time="2019-08-28T23:37:42.570893165Z" level=info msg="Daemon has completed initialization"
time="2019-08-28T23:37:42.593056000Z" level=info msg="API listen on /var/run/docker.sock"
```

Also, `docker ps` should work fine:

```bash
root@sc:/# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

Once Docker is running inside the system container, use it as usual.
For example, to start an (inner) busybox container:

```bash
root@sc:/# docker run -it --hostname my_inner_cont busybox
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
ee153a04d683: Pull complete
Digest: sha256:9f1003c480699be56815db0f8146ad2e22efea85129b5b5983d0e0fb52d9ab70
Status: Downloaded newer image for busybox:latest
/ #
```

As you can see, the system container is running Docker inside of it,
with total isolation from the host's Docker daemon.

The Dockerfiles for the images in the Nestybox repo are
[here](../dockerfiles/).

Feel free to source the Nestybox sample images from your own Dockerfile,
or make a copy of a Nestybox Dockerfile and modify it per your needs.
Instructions for doing so are [here](../dockerfiles/README.md).

### Inner & Outer Containers

When launching Docker inside a system container, terminology can
quickly get confusing due to container nesting.

To prevent confusion we refer to the containers as the "outer" and
"inner" containers.

* The outer container is a system container, created at the host
  level; it's launched with Docker + Sysboxd.

* The inner container is an application container, created within
  the outer container. It's launched by the Docker instance running
  inside the outer container.

### Inner Docker Image Caching

The Docker instance running inside the system container stores its
images in the `/var/lib/docker` directory inside the container.

When the system container is removed, the contents of that directory
will also be removed.

If you wish to keep the contents of that directory so that they may be
reused by a future system container instance (e.g., to avoid forcing
the future instance to re-download inner container images), then
simply mount a Docker volume on the host into the system container's
`/var/lib/docker`:

```bash
$ docker volume create myVol
$ docker run --runtime=sysbox-runc -it --mount source=myVol,target=/var/lib/docker debian:latest
```

This way, the inner Docker's image cache will persist even after the
system container is removed.

But note the following: Docker does not support two or more daemons
sharing the same image cache. Thus, if you follow the approach above,
you must mount the host volume to a **single** system container at any
given time.

If you wish to have multiple system containers using this technique,
use a separate host volume for each:

```bash
$ docker volume create myVol1
$ docker volume create myVol2
$ docker run --runtime=sysbox-runc -it --mount source=myVol1,target=/var/lib/docker debian:latest
$ docker run --runtime=sysbox-runc -it --mount source=myVol2,target=/var/lib/docker debian:latest
```

### Inner Docker Restrictions

The Docker instance inside the system container is assumed to store
it's images at the usual `/var/lib/docker`. This is known as the
Docker "data-root".

While it's possible to configure the inner Docker to store it's images
at some other location within the system container (via the Docker
daemon's `--data-root` option), Sysboxd does **not** currently support
this (i.e., the inner Docker won't work).

## Sysboxd Reconfiguration

The Sysboxd installer starts the [Sysboxd components](design.md#sysboxd-components)
automatically, using systemd.

Normally this is sufficient and the user need not worry about reconfiguring Sysboxd.

However, there are scenarios where the daemons may need to be
reconfigured (e.g., to enable a given option on sysbox-fs or
sysbox-mgr). For example, increasing the log level by passing the
`--log-level debug` to the sysbox-fs or sysbox-mgr daemons.

In these cases, do the following:

1) Modify the desired service initialization instruction.

   Example:

   ```bash
   $ sudo sed -i '/^ExecStart/ s/$/ --log-level debug/' /etc/systemd/system/sysboxd.service.wants/sysbox-fs.service
   $
   $ egrep "ExecStart" /etc/systemd/system/sysboxd.service.wants/sysbox-fs.service
   ExecStart=/usr/local/sbin/sysbox-fs --log /var/log/sysbox-fs.log --log-level debug
   ```

2) Reload **systemd** to digest the previous change:

   ```bash
   $ sudo systemctl daemon-reload
   ```

3) Restart the **sysboxd** service:

   ```bash
   $ sudo systemctl restart sysboxd
   ```

Note that even though Sysboxd is comprised of various daemons and its
respective services, you should only interact with its outer-most
systemd service: **sysboxd**.

## Rootless Container Support

Sysboxd must run with root privileges on the host system. It won't
work if executed without root privileges.

## Interaction with Docker Userns Remap

Docker has a configuration called [userns-remap](https://docs.docker.com/engine/security/userns-remap)
that enables the Linux user namespace in containers. By default, userns-remap is disabled in Docker.

Sysboxd works out-of-the-box with either configuration of Docker
(i.e., with or without userns-remap enabled). No change to Sysboxd's
configuration is needed either way.

However, Sysboxd works best with Docker userns-remap *disabled*
(though this is supported by Sysboxd in a reduced number of distros,
as described [here](../README.md#supported-linux-distros)).

The reason userns-remap disabled is preferred is that in this case
Sysboxd will allocate exclusive user-ID and group-ID mappings for the
system container's user namespace, thereby improving
isolation between system containers.

If on the other hand Docker userns-remap is enabled, then Docker
chooses the user-ID and group-ID mappings for the container and
Sysboxd honors these. However Docker currently has a limitation: it
uses the same user-ID and group-ID mappings for all containers,
thereby decreasing isolation between containers (i.e., if a process
escapes the container, it may be able to access the filesystem
of other containers).

The table below summarizes this:

| Docker userns-remap | Description |
|---------------------|-------------|
| disabled            | sysboxd will allocate exclusive uid(gid) mappings per system container and perform uid(gid) shifting. |
|                     | Strong container-to-host isolation. |
|                     | Strong container-to-container isolation. |
|                     | Uses the Nestybox Shiftfs module in the kernel. |
|                     |
| enabled             | sysboxd will honor Docker's uid(gid) mappings. |
|                     | Strong container-to-host isolation. |
|                     | Reduced container-to-container isolation (same uid(gid) range). |
|                     | Does not use the Nestybox Shiftfs module in the kernel. |

For further info Sysboxd's usage of the Linux user namespace and
associated ID mappings, refer to the [Sysboxd design document](design.md).

## Docker Bind Mount Permissions

Sysboxd system containers support all Docker storage mount types:
volume, bind, or tmpfs.

However, for bind mounts there are some caveats. These caveats do not apply to
volume mounts and tmpfs mounts.

The caveats for bind mounts arise from the fact that system containers
use the Linux user namespace, and therefore the root user inside the
system container maps to a non-root user in the host.

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
$ docker run --runtime=sysbox-runc -it --mount type=bind,source=/a/b/c,target=/mnt/somedir debian:latest
```

the bind mount source path at `/a/b/c` must be accesible by root only. This
means that at least one of `/a` or `/a/b` or `/a/b/c` must have `0700`
permissions on the host.

Note that if either `/a` or `/a/b` have root ownership with `0700`
permissions, then the last directory in the path (`c`) could have less
restrictive permissions (e.g., `0644` or even `0777`), although it
must be owned by `root:root`.

The rationale for these rules is the following.

Rule (1) ensures that the root user in the system container will have
correct permissions to access the bind mount. Sysboxd uses the
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

## Support for Docker Volume Plugins

Docker supports a number of [volume plugins](https://docs.docker.com/engine/extend/legacy_plugins/).
These extend Docker's default volume support and allow storing a
container's data across a cluster of hosts, in the cloud, etc.

Nestybox does not yet support using any of these plugins
for system containers.
