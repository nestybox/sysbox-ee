Sysbox User's Guide
====================

The Sysbox [README](../README.md) file contains the basic information
on how to install Sysbox and create system containers with it. This
document supplements the README file with additional information.

## Running System Containers with Sysbox

We currently support two ways of running system containers with Sysbox:

1) Using Docker (the easy and preferred way)

2) Using the `sysbox-runc` command directly (more control but harder)

### Using Docker

It's easy to run system container using Docker. Simply use the `--runtime=sysbox-runc`
flag in the `docker run` command:

```console
$ docker run --runtime=sysbox-runc --rm -it --hostname my_cont debian:latest
root@my_cont:/#
```

It's possible to configure Sysbox as the default runtime for Docker. This
way you don't have to use the `--runtime` flag everytime. If you wish to do this,
refer to the [Docker website](https://docs.docker.com/engine/reference/commandline/dockerd/).

### Using the sysbox-runc command

It's also possible to launch system containers directly via the
`sysbox-runc` command.

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

### Support for Other Container Managers

We officially only support the above methods to run Sysbox.

However, we plan to add support for other OCI compatible container
managers (e.g., [cri-o](https://cri-o.io/)) soon.

## Running Software inside the System Container

A system container is logically a super-set of a regular Docker
application container, and thus should be able to run any application
that runs in a regular Docker container plus system-level software
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

You can find a few such images in the [Nestybox DockerHub repo](https://hub.docker.com/r/nestybox).

For example, to run a system container that has Ubuntu Disco + Docker inside, simply
type:

```console
$ docker run --runtime=sysbox-runc -it --hostname sc nestybox/ubuntu-disco-docker:latest
root@sc:/#
```

From within the system container, start Docker:

```console
root@sc:/# dockerd > /var/log/dockerd.log 2>&1 &
```

Note that we don't yet support systemd inside the system container, so
the Docker daemon needs to be started manually as shown above.

The Docker daemon should now be running inside the system container.
You can verify this by looking at the Docker daemon log file:

```console
root@sc:/# tail -n 2 /var/log/dockerd.log
time="2019-08-28T23:37:42.570893165Z" level=info msg="Daemon has completed initialization"
time="2019-08-28T23:37:42.593056000Z" level=info msg="API listen on /var/run/docker.sock"
```

Also, `docker ps` should work fine:

```console
root@sc:/# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

Once Docker is running inside the system container, use it as usual.
For example, to start an (inner) busybox container:

```console
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

The [Nestybox blog site](https://blog.nestybox.com) has more info on
Docker-in-Docker and it's use cases.

### Inner & Outer Containers

When launching Docker inside a system container, terminology can
quickly get confusing due to container nesting.

To prevent confusion we refer to the containers as the "outer" and
"inner" containers.

* The outer container is a system container, created at the host
  level; it's launched with Docker + Sysbox.

* The inner container is an application container, created within
  the outer container. It's launched by the Docker instance running
  inside the outer container.

### Inner Docker Restrictions

The Docker instance inside the system container is assumed to store
it's images at the usual `/var/lib/docker`. This is known as the
Docker "data-root".

While it's possible to configure the inner Docker to store it's images
at some other location within the system container (via the Docker
daemon's `--data-root` option), Sysbox does **not** currently support
this (i.e., the inner Docker won't work).

### Inner Docker Image Caching

The Docker instance running inside the system container stores its
images in the `/var/lib/docker` directory inside the container.

When the system container is removed (i.e., not just stopped, but
actualy removed via `docker rm`), the contents of that directory will
also be removed.

This means that the inner Docker's image cache is destroyed when the
associated system container is removed.

It's possible to override this behavior by mounting a host volume into
the system container's `/var/lib/docker`, in order to persist the
inner Docker's image cache accross system container lifecycles:

```console
$ docker run --runtime=sysbox-runc -it --hostname sc --mount type=bind,source=/some/host/dir,target=/var/lib/docker nestybox/ubuntu-disco-docker:latest
```

However, Sysbox does not currently support mounts into the system
container's `/var/lib/docker` when Docker is configured with docker
userns-remap disabled (the mount is in fact ignored in this
case). This restriction is due to a low-level problem in the
interaction between overlayfs and Ubuntu's `shiftfs` module, which is
expected to be resolved soon.

## Sysbox Reconfiguration

The Sysbox installer starts the [Sysbox components](design.md#sysbox-components)
automatically, using systemd.

Normally this is sufficient and the user need not worry about reconfiguring Sysbox.

However, there are scenarios where the daemons may need to be
reconfigured (e.g., to enable a given option on sysbox-fs or
sysbox-mgr). For example, increasing the log level by passing the
`--log-level debug` to the sysbox-fs or sysbox-mgr daemons.

In these cases, do the following:

1) Modify the desired service initialization instruction.

   Example:

```console
$ sudo sed -i '/^ExecStart/ s/$/ --log-level debug/' /etc/systemd/system/sysbox.service.wants/sysbox-fs.service
$
$ egrep "ExecStart" /etc/systemd/system/sysbox.service.wants/sysbox-fs.service
ExecStart=/usr/local/sbin/sysbox-fs --log /var/log/sysbox-fs.log --log-level debug
```

2) Reload **systemd** to digest the previous change:

```console
$ sudo systemctl daemon-reload
```

3) Restart the **sysbox** service:

```console
$ sudo systemctl restart sysbox
```

Note that even though Sysbox is comprised of various daemons and its
respective services, you should only interact with its outer-most
systemd service: **sysbox**.

## Rootless Container Support

Sysbox must run with root privileges on the host system. It won't
work if executed without root privileges.

## Interaction with Docker Userns Remap

Docker has a configuration called [userns-remap](https://docs.docker.com/engine/security/userns-remap)
that enables the Linux user namespace in containers. By default, userns-remap is disabled in Docker.

Sysbox works out-of-the-box with either configuration of Docker
(i.e., with or without userns-remap enabled). No change to Sysbox's
configuration is needed either way.

However, Sysbox works best with Docker userns-remap *disabled*
(though this reduces the number of [distros on which Sysbox runs](../README.md#supported-linux-distros)).

The reason userns-remap disabled is preferred is that in this case
Sysbox will allocate exclusive user-ID and group-ID mappings for the
system container's user namespace, thereby improving
isolation between system containers.

If on the other hand Docker userns-remap is enabled, then Docker
chooses the user-ID and group-ID mappings for the container and
Sysbox honors these. However Docker currently has a limitation: it
uses the same user-ID and group-ID mappings for all containers,
thereby decreasing isolation between containers (i.e., if a process
escapes the container, it may be able to access the filesystem
of other containers).

The table below summarizes this:

| Docker userns-remap | Description |
|---------------------|-------------|
| Disabled            | Sysbox will allocate exclusive uid(gid) mappings per system container and perform uid(gid) shifting. |
|                     | Strong container-to-host isolation. |
|                     | Strong container-to-container isolation. |
|                     | Uses the Ubuntu shiftfs module in the kernel. |
|                     |
| Enabled             | Sysbox will honor Docker's uid(gid) mappings. |
|                     | Strong container-to-host isolation. |
|                     | Reduced container-to-container isolation (same uid(gid) range). |
|                     | Does not use the Ubuntu shiftfs module in the kernel. |

For further info Sysbox's usage of the Linux user namespace and
associated ID mappings, refer to the [Sysbox design document](design.md).

## Docker Bind Mount Requirements

Sysbox system containers support all Docker storage mount types:
[volume, bind, or tmpfs](https://docs.docker.com/storage/).

However, for bind mounts there are some important requirements in
order to deal with file ownership issues. These vary depending on
whether Docker is configured with userns-remap or not, as explained
below.

### Without Docker userns-remap

This is the default Docker configuration.

In this case, Sysbox will automatically mount the Ubuntu shiftfs
filesystem on the bind-mount source when a container starts and the
mount will persist until the container is stopped.

Sysbox uses the shiftfs mount to ensure that processes inside the
container see the right ownership for the contents of the bind mounted
directory as described [here](design.md/#ubuntu-shiftfs-module).

As a result, the following requirements apply to system container bind
mounts:

* Files in the bind mounted directory (or the bind mounted file itself)
  should be owned by users in the range [0:65535]. The used-ID and group-ID
  of these files will be mapped inside the system container as follows:

  | File Ownership on Host | File Ownership in System Container |
  |------------------------|------------------------------------|
  | 0 (root) -> 65535      | 0 (root) -> 65535                  |
  | Others                 | nobody:nogroup                     |

  Note that it's possible for multiple system containers to share a
  bind-mount, as all system containers will use the file ownership
  mapping shown above.

* To improve security, one or more of the following requirements
  should be in place for the bind mount source file or directory:

  - It's accessible only by host's root user (i.e., `0700` permission
    somewhere in the path).

  - It's mounted into the system container as "read-only" when possible:

```console
$ docker run --runtime=sysbox-runc -it --mount type=bind,source=/path/to/bind/source,target=/path/to/mnt/point,readonly my-syscont
```

  - It's re-mounted on the host with the `noexec` attribute prior to
    being bind mounted into the system container:

```console
$ sudo mount --bind /path/to/bind/source /path/to/bind/source
$ sudo mount -o remount,bind,noexec /path/to/bind/source /path/to/bind/source
$ docker run --runtime=sysbox-runc -it --mount type=bind,source=/path/to/bind/source,target=/path/to/mnt/point my-syscont`
```

  These are not hard requirements, meaning that if they aren't met the
  bind mount into the system container will still work. However,
  failure to meet one of these requirements reduces host security
  as described [here](design.md#shiftfs-security-precautions).

### With Docker userns-remap

If Docker is configured with userns-remap, then the following
rule applies to bind mounts:

* Files in the bind mounted directory (or the bind mounted file
  itself) should be owned by users in the range [uid:uid+65535], where
  `uid` is the host's user-ID associated with the docker userns-remap
  configuration. The same applies to group-IDs.

For example, if the docker userns-remap configuration is the following:

```console
$ cat /etc/docker/daemon.json
{
   "userns-remap": "someuser"
}
```

then the bind mount source directory and/or files must be owned by the
subuid(gid) range associated with `someuser`. That range can be found in
`/etc/subuid` and `/etc/subgid`. This will ensure that the file shows
up with appropriate ownership inside the system container.

For example:

```console
$ cat /etc/subuid
someuser:100000:65536
sysbox:165536:268435456

$ cat /etc/subgid
someuser:100000:65536
sysbox:165536:268435456
```

Then the bind mount source should be owned by `100000:100000` because that's
the start of the subuid(gid) range associated with `someuser`.

Note that the security requirements listed in the prior section don't
apply when Docker is configured with userns-remap. That's because
files written from within the system container will have ownership
corresponding to the subuid(gid) associated with user `someuser`,
rather than `root:root`.

## Storage sharing between system containers

System containers use the Linux user-namespace for increased isolation
from the host and from other containers (i.e., each container has a
range of uids(gids) that map to a non-root user in the host).

A known issue with containers that use the user-namespace is that
sharing storage between them is not trivial because each container may
be assigned an exclusive uid(gid) range on the host, and thus may not
have access to the shared storage (unless such storage has lax
permissions).

Sysbox system containers support storage sharing between multiple
system containers, without lax permissions and in spite of the fact
that each system container may be assigned an exclusive uid/gid range
on the host.

This is possible due to the uid(gid) shifting performed by the Ubuntu
`shiftfs` module.

Setting it up is simple:

First, create a shared directory owned by `root:root`:

```console
$ sudo mkdir <path/to/shared/dir>
```

Then simply bind-mount the directory into the system container(s):

```console
$ docker run --runtime=sysbox-runc --rm -it --hostname syscont --mount type=bind,source=<path/to/shared/dir>,target=</mount/path/inside/container> debian:latest
```

When the system container is launched this way, Sysbox will notice
that bind mounted directory is owned by `root:root` and will mount
shiftfs on top of it, such that the container can have access to
it. Repeating this for multiple system containers will give all of
them access to the shared direcotry.

Note: for improve host security on the bind mount source, follow the
recommendations described [above](#docker-bind-mount-requirements).

## Support for Linux Security Modules (LSMs)

### AppArmor

Sysbox currently uses a permissive AppArmor profile for system
containers. We plan to create a more restrictive profile in
the near future.

### SELinux

Sysbox does not yet support running on systems with SELinux enabled.

### Others

Sysbox does not have support for other Linux LSMs at this time.
