# Sysbox Quick Start Guide

This document is a quick guide showing how to deploy system containers
with Sysbox and take advantage of their features.

The document is primarily composed of examples, with quick explanatory
text in between. For a more thorough explanation, refer to the
[Sysbox User's Guide](usage.md).

## Contents

-   [Sysbox Installation](#sysbox-installation)
-   [Deploy a System Container](#deploy-a-system-container)
-   [Nestybox Image Repository](#nestybox-image-repository)
-   [Deploy a System Container with Docker inside](#deploy-a-system-container-with-docker-inside)
    -   [Inner and Outer Containers](#inner-and-outer-containers)
-   [Deploy a System Container with Systemd inside](#deploy-a-system-container-with-systemd-inside)
-   [Deploy a System Container with Systemd, sshd, and Docker inside](#deploy-a-system-container-with-systemd-sshd-and-docker-inside)
-   [Deploy a System Container with Supervisord and Docker inside](#deploy-a-system-container-with-supervisord-and-docker-inside)
-   [Building A System Container That Includes Inner Container Images](#building-a-system-container-that-includes-inner-container-images)
-   [Committing A System Container That Includes Inner Container Images](#committing-a-system-container-that-includes-inner-container-images)
-   [Persistence of Inner Container Images with Docker Volumes](#persistence-of-inner-container-images-with-docker-volumes)
-   [Persistence of Inner Container Images with Bind Mounts](#persistence-of-inner-container-images-with-bind-mounts)
-   [Sharing Storage Among System Containers](#sharing-storage-among-system-containers)
-   [System Container Isolation Features](#system-container-isolation-features)
-   [Sysbox Uninstallation](#sysbox-uninstallation)
-   [Further reading](#further-reading)

## Sysbox Installation

Refer to the [Sysbox README file](../README.md) for the supported
Linux distros, host requirements, and installation instructions.

## Deploy a System Container

The easiest way is to use Docker; simply add the `--runtime` flag to `docker run`:

```console
$ docker run --runtime=sysbox-runc --rm -it --hostname syscont debian:latest
root@syscont:/#
```

Inside the system container you can now deploy system-level software
(e.g., Docker, systemd) just as you would on a real host or VM.

Later sections in this document show examples of this.

It's also possible to deploy a system container without Docker, using
the `sysbox-runc` command.  See [this section](usage.md#running-system-containers-with-sysbox)
in the Sysbox User's Guide for more info.

## Nestybox Image Repository

The [Nestybox Docker Hub](https://hub.docker.com/u/nestybox) site has several reference system container images.

The Dockerfiles for those can be found [here](../dockerfiles).

Feel free to source the Nestybox sample images from your own Dockerfile,
or make a copy of a Nestybox Dockerfile and modify it per your needs.
Instructions for doing so are [here](../dockerfiles/README.md).

## Deploy a System Container with Docker inside

We will use a system container image that has Alpine + Docker inside. It's called
`nestybox/alpine-docker` and it's in the Nestybox DockerHub public repo. The
Dockerfile is [here](../dockerfiles/alpine-docker/Dockerfile).

```console
$ docker run --runtime=sysbox-runc -it --hostname=syscont nestybox/alpine-docker:latest

/ # which docker
/usr/bin/docker

/ # dockerd > /var/log/dockerd.log 2>&1 &

/ # tail /var/log/dockerd.log
time="2019-10-23T20:48:51.960846074Z" level=warning msg="Your kernel does not support cgroup rt runtime"
time="2019-10-23T20:48:51.960860148Z" level=warning msg="Your kernel does not support cgroup blkio weight"
time="2019-10-23T20:48:51.960872060Z" level=warning msg="Your kernel does not support cgroup blkio weight_device"
time="2019-10-23T20:48:52.146157113Z" level=info msg="Loading containers: start."
time="2019-10-23T20:48:52.235036055Z" level=info msg="Default bridge (docker0) is assigned with an IP address 172.18.0.0/16. Daemon option --bip can be used to set a preferred IP address"
time="2019-10-23T20:48:52.324207525Z" level=info msg="Loading containers: done."
time="2019-10-23T20:48:52.476235437Z" level=warning msg="Not using native diff for overlay2, this may cause degraded performance for building images: failed to set opaque flag on middle layer: operation not permitted" storage-driver=overlay2
time="2019-10-23T20:48:52.476418516Z" level=info msg="Docker daemon" commit=0dd43dd87fd530113bf44c9bba9ad8b20ce4637f graphdriver(s)=overlay2 version=18.09.8-ce
time="2019-10-23T20:48:52.476533826Z" level=info msg="Daemon has completed initialization"
time="2019-10-23T20:48:52.489489309Z" level=info msg="API listen on /var/run/docker.sock"

/ # docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES

/ # docker run -it busybox
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
7c9d20b9b6cd: Pull complete
Digest: sha256:fe301db49df08c384001ed752dff6d52b4305a73a7f608f21528048e8a08b51e
Status: Downloaded newer image for busybox:latest
/ #
```

As shown, Docker runs normally inside the secure system container and we
can deploy an inner container (busybox) without problem.

The Sysbox runtime allows you to do this without resorting to an
unsecure Docker privileged container.

### Inner and Outer Containers

When launching Docker inside a system container, terminology can
quickly get confusing due to container nesting.

To prevent confusion we refer to the containers as the "outer" and
"inner" containers.

-   The outer container is a system container, created at the host
    level; it's launched with Docker + Sysbox.

-   The inner container is an application container, created within the
    outer container (i.e., it's created by the Docker instance running
    inside the system container (aka the inner Docker)).

## Deploy a System Container with Systemd inside

Deploying systemd inside a system container is useful when you plan to
run multiple services inside the system container, or when you want to
use it as a virtual host environment.

We will use a system container image that has Ubuntu Bionic + Systemd
inside. It's called `nestybox/ubuntu-bionic-systemd` and it's in the
Nestybox DockerHub public repo. The Dockerfile is [here](../dockerfiles/ubuntu-bionic-systemd/Dockerfile).

```console
$ docker run --runtime=sysbox-runc --rm -it --hostname=syscont nestybox/ubuntu-bionic-systemd
systemd 237 running in system mode. (+PAM +AUDIT +SELINUX +IMA +APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 +SECCOMP +BLKID +ELFUTILS +KMOD -IDN2 +IDN -PCRE2 default-hierarchy=hybrid)
Detected virtualization container-other.
Detected architecture x86-64.

Welcome to Ubuntu 18.04.3 LTS!

Set hostname to <syscont>.
Failed to read AF_UNIX datagram queue length, ignoring: No such file or directory
Failed to install release agent, ignoring: No such file or directory
File /lib/systemd/system/systemd-journald.service:35 configures an IP firewall (IPAddressDeny=any), but the local system does not support BPF/cgroup based firewalling.
Proceeding WITHOUT firewalling in effect! (This warning is only shown for the first loaded unit using IP firewalling.)
[  OK  ] Reached target Swap.

...

[  OK  ] Reached target Login Prompts.
[  OK  ] Started Login Service.
[  OK  ] Reached target Multi-User System.
[  OK  ] Reached target Graphical Interface.
         Starting Update UTMP about System Runlevel Changes...
[  OK  ] Started Update UTMP about System Runlevel Changes.

Ubuntu 18.04.3 LTS syscont console

syscont login:
```

In the system container image we are using, we've configured the
default console login and password to be `admin/admin` (you can always
change this in the image's Dockerfile). Let's login with these
credentials:

```console
syscont login: admin
Password:
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 5.0.0-31-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

admin@syscont:~$
```

And verify systemd is running correctly:

```console
admin@syscont:~$ ps -fu root
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 23:41 ?        00:00:00 /sbin/init
root       252     1  0 23:41 ?        00:00:00 /lib/systemd/systemd-journald
root       685     1  0 23:41 ?        00:00:00 /lib/systemd/systemd-logind
root       725     1  0 23:41 pts/0    00:00:00 /bin/login -p --

admin@syscont:~$ systemctl
UNIT                                     LOAD   ACTIVE SUB       DESCRIPTION
-.mount                                  loaded active mounted   Root Mount
dev-full.mount                           loaded active mounted   /dev/full
dev-kmsg.mount                           loaded active mounted   /dev/kmsg
dev-mqueue.mount                         loaded active mounted   POSIX Message Queue File System
dev-null.mount                           loaded active mounted   /dev/null
dev-random.mount                         loaded active mounted   /dev/random

...

timers.target                            loaded active active    Timers
apt-daily-upgrade.timer                  loaded active waiting   Daily apt upgrade and clean activities
apt-daily.timer                          loaded active waiting   Daily apt download activities
motd-news.timer                          loaded active waiting   Message of the Day
systemd-tmpfiles-clean.timer             loaded active waiting   Daily Cleanup of Temporary Directories

LOAD   = Reflects whether the unit definition was properly loaded.
ACTIVE = The high-level unit activation state, i.e. generalization of SUB.
SUB    = The low-level unit activation state, values depend on unit type.

74 loaded units listed. Pass --all to see loaded but inactive units, too.
To show all installed unit files use 'systemctl list-unit-files'.
```

To exit the system container, you can break from it (by pressing
`ctrl-p ctrl-q`). You can then stop the system container by using the
`docker stop` command from the host.

Alternatively, from another shell type:

```console
$ docker ps
CONTAINER ID        IMAGE                            COMMAND             CREATED             STATUS              PORTS               NAMES
3236bcdd2313        nestybox/ubuntu-bionic-systemd   "/sbin/init"        23 minutes ago      Up 23 minutes                           zen_blackburn

$ docker stop zen_blackburn
zen_blackburn
```

And back in the shell where the system container is running, you'll
see systemd shutting down all services in the system container:

```console
[  OK  ] Removed slice system-getty.slice.
[  OK  ] Stopped target Host and Network Name Lookups.
         Stopping Network Name Resolution...
[  OK  ] Stopped target Graphical Interface.
[  OK  ] Stopped target Multi-User System.

...

[  OK  ] Reached target Shutdown.
[  OK  ] Reached target Final Step.
         Starting Halt...
```

## Deploy a System Container with Systemd, sshd, and Docker inside

Earlier we showed an example of deploying a system container that has
Docker inside. In that earlier example we did not have Systemd (or any
other process manager) in the container, so we had to manually start
Docker inside the container.

This example improves on this by deploying a system container that
has both Systemd and Docker inside. You'll see that as soon as the
system container is started, the Docker daemon inside the system container
is ready to be used.

Further, we've also added an SSH daemon in into the system container image,
so that you can login remotely into it, just as you would on a physical
host or VM.

We will use a system container image called `nestybox/ubuntu-bionic-systemd-docker:latest` which is in
Nestybox DockerHub public repo. The Dockerfile is [here](../dockerfiles/ubuntu-bionic-systemd-docker/Dockerfile).

```console
$ docker run --runtime=sysbox-runc -it --rm -P --hostname=syscont nestybox/ubuntu-bionic-systemd-docker:latest
systemd 237 running in system mode. (+PAM +AUDIT +SELINUX +IMA +APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 +SECCOMP +BLKID +ELFUTILS +KMOD -IDN2 +IDN -PCRE2 default-hierarchy=hybrid)
Detected virtualization container-other.
Detected architecture x86-64.

Welcome to Ubuntu 18.04.3 LTS!

Set hostname to <syscont>.

...

[  OK  ] Started Docker Application Container Engine.
[  OK  ] Reached target Multi-User System.
[  OK  ] Reached target Graphical Interface.
         Starting Update UTMP about System Runlevel Changes...
[  OK  ] Started Update UTMP about System Runlevel Changes.

Ubuntu 18.04.3 LTS syscont console

syscont login:
```

In the system container image we are using, we've configured the
default console login and password to be `admin/admin`. You can always
change this in the image's Dockerfile.

```console
syscont login: admin
Password:
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 5.0.0-31-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

admin@syscont:~$
```

Now verify that Docker is running inside the system container:

```console
admin@syscont:~$ systemctl status docker.service
● docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2019-10-24 00:33:09 UTC; 8s ago
     Docs: https://docs.docker.com
 Main PID: 715 (dockerd)
    Tasks: 12
   CGroup: /system.slice/docker.service
           └─715 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

admin@syscont:~$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

And run an inner container:

```console
admin@syscont:~$ docker run -it busybox
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
7c9d20b9b6cd: Pull complete
Digest: sha256:fe301db49df08c384001ed752dff6d52b4305a73a7f608f21528048e8a08b51e
Status: Downloaded newer image for busybox:latest
/ #
```

Now let's ssh into the system container. In order to do this, we need
the host's IP address as well as the host port that is mapped to the
system container's sshd port.

In my case, the host's IP address is 10.0.0.230. The ssh daemon is
listening on port 22 in the system container, which is mapped to some
arbitrary port on the host machine.

Let's find out what that arbitrary port is. From the host, type:

```console
$ docker ps
CONTAINER ID        IMAGE                                          COMMAND             CREATED             STATUS              PORTS                   NAMES
e22773df703e        nestybox/ubuntu-bionic-systemd-docker:latest   "/sbin/init"        16 seconds ago      Up 15 seconds       0.0.0.0:32770->22/tcp   sad_kepler
```

Now let's ssh into the system container from a different machine:

```console
$ ssh admin@10.0.0.230 -p 32770

The authenticity of host '[10.0.0.230]:32770 ([10.0.0.230]:32770)' can't be established.
ECDSA key fingerprint is SHA256:VNHrxvsHp4aJYH/DQjvBMdeoF0HBP2yKtWc815WtnnI.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[10.0.0.230]:32770' (ECDSA) to the list of known hosts.
admin@10.0.0.230's password:
Last login: Thu Oct 24 03:47:39 2019
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

admin@syscont:~$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

The ssh worked without problem.

This is cool because you now have a system container that is acting
like a virtual host with Systemd and sshd. Plus it has Docker inside
so you can deploy application containers in complete isolation from
the underlying host.

## Deploy a System Container with Supervisord and Docker inside

Systemd is great but may be too heavy-weight for some use cases.

A good alternative is to use supervisord as a light weight process
manager inside a system container.

We will use a system container image called `nestybox/alpine-supervisord-docker:latest`.
Nestybox DockerHub public repo. The Dockerfile, supervisord.conf, and docker-entrypoint.sh files
can be found [here](../dockerfiles/alpine-supervisord-docker).

```console
$ docker run --runtime=sysbox-runc -d --rm -P --hostname=syscont nestybox/alpine-supervisord-docker:latest
f3b90976ad0550fc8142568d988c8fa65c54864d04c1637e88323a32f87cf3af
```

Let's check that all services inside the system container were started
correctly. From the host, type:

```console
$ docker ps
CONTAINER ID        IMAGE                                       COMMAND                  CREATED             STATUS              PORTS                   NAMES
f3b90976ad05        nestybox/alpine-supervisord-docker:latest   "/usr/bin/docker-ent…"   2 seconds ago       Up 1 second         0.0.0.0:32776->22/tcp   sleepy_shamir

$ docker exec -it sleepy_shamir ps
PID   USER     TIME  COMMAND
    1 root      0:00 {supervisord} /usr/bin/python2 /usr/bin/supervisord -n
    7 root      0:00 /usr/sbin/sshd -D
    8 root      0:02 /usr/bin/dockerd
   36 root      0:03 containerd --config /var/run/docker/containerd/containerd.
  980 root      0:00 ps
```

As shown, supervisord is running at the init process and has spawned
sshd and dockerd. Cool.

Now let's ssh into the system container. In this example the host
machine is at IP 10.0.0.230, and the system container's ssh port is
mapped to host port 32776 as indicated by the `docker ps` output
above. The login is `root:root` as configured in the image's Dockerfile.

```console
$ ssh root@10.0.0.230 -p 32776
The authenticity of host '[10.0.0.230]:32776 ([10.0.0.230]:32776)' can't be established.
RSA key fingerprint is SHA256:/p++Ju2yo5SF1obEV4TeI+Fq6Q2DBErdboO287aSNp0.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[10.0.0.230]:32776' (RSA) to the list of known hosts.
root@10.0.0.230's password:
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <http://wiki.alpinelinux.org/>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.
syscont:~#
```

Now run a Docker container inside the system container:

```console
syscont:~# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
syscont:~# docker run -it busybox
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
7c9d20b9b6cd: Pull complete
Digest: sha256:fe301db49df08c384001ed752dff6d52b4305a73a7f608f21528048e8a08b51e
Status: Downloaded newer image for busybox:latest
/ # syscont:~#
```

## Building A System Container That Includes Inner Container Images

One of the novel features of Sysbox is that it allows you to use Docker
to create a system container image that includes one or more inner Docker
container images within it.

This way, you can deploy the system container image and deploy it's
inner containers without having to pull the inner container images
from the network.

There are two ways to create such a system container image.
Via the `docker build` command or via the `docker commit` command.

This section shows an example with `docker build`. The [next section](#committing-a-system-container-that-includes-inner-container-images)
shows an example with `docker commit`.

In order to use the `docker build` command, we must first reconfigure
the host's Docker daemon to use the `sysbox-runc` runtime as it's
default runtime.

This is needed because the Docker build process must call into the
Sysbox runtime and unfortunately the `docker build` command does not
currently take a `--runtime` option (unlike `docker run`).

To reconfigure the Docker daemon, we edit its
`/etc/docker/daemon.json` file.  Here is how the
`/etc/docker/daemon.json` file should look like. Notice the line at
the end of the file:

```console
# more /etc/docker/daemon.json
{
    "runtimes": {
        "sysbox-runc": {
            "path": "/usr/local/sbin/sysbox-runc"
        }
    },
    "default-runtime": "sysbox-runc"
}
```

We then restart the Docker daemon service:

```console
$ systemctl restart docker.service
```

Once we've configured sysbox-runc as the Docker default runtime, we
can now build the system container image.

Here is a sample Dockerfile:

```dockerfile
FROM nestybox/alpine-docker

COPY docker-pull.sh /usr/bin
RUN chmod +x /usr/bin/docker-pull.sh && docker-pull.sh && rm /usr/bin/docker-pull.sh
```

This Dockerfile inherits from the `nestybox/alpine-docker` base image which simply contains
Alpine plus a Docker daemon (the Dockerfile is [here](../dockerfiles/alpine-docker/Dockerfile)).

The presence of the Docker daemon in the base image is required since
we will use it to pull the inner container images.

The key instruction in the Dockerfile shown above is the `RUN`
instruction. Notice that it's copying a script called `docker-pull.sh`
into the system container, executing it, and removing it.

The `docker-pull.sh` script is shown below.

```bash
#!/bin/sh

# dockerd start
dockerd > /var/log/dockerd.log 2>&1 &
sleep 2

# pull inner images
docker pull busybox:latest
docker pull alpine:latest

# dockerd cleanup (remove the .pid file as otherwise it prevents
# dockerd from launching correctly inside sys container)
kill $(cat /var/run/docker.pid)
kill $(cat /run/docker/containerd/containerd.pid)
rm -f /var/run/docker.pid
rm -f /run/docker/containerd/containerd.pid
```

As shown, the script simply runs Docker inside the system container,
pulls the inner container images (in this case the busybox and alpine
images), and does some cleanup. Pretty simple.

The reason we need this script in the first place is because it's hard
to put all of these commands into a single Dockerfile `RUN`
instruction. It's simpler to put them in a separate script and call
it from the `RUN` instruction.

Let's see what happens when we execute `docker build` on this Dockerfile
to build the system container image:

```console
$ docker build -t nestybox/syscont-with-inner-containers:latest .

Sending build context to Docker daemon  3.072kB
Step 1/3 : FROM nestybox/alpine-docker
 ---> b51716d05554
Step 2/3 : COPY docker-pull.sh /usr/bin
 ---> Using cache
 ---> df2af1f26937
Step 3/3 : RUN chmod +x /usr/bin/docker-pull.sh && docker-pull.sh && rm /usr/bin/docker-pull.sh
 ---> Running in 7fa2687f2385
latest: Pulling from library/busybox
7c9d20b9b6cd: Pulling fs layer
7c9d20b9b6cd: Verifying Checksum
7c9d20b9b6cd: Download complete
7c9d20b9b6cd: Pull complete
Digest: sha256:fe301db49df08c384001ed752dff6d52b4305a73a7f608f21528048e8a08b51e
Status: Downloaded newer image for busybox:latest
latest: Pulling from library/alpine
89d9c30c1d48: Pulling fs layer
89d9c30c1d48: Verifying Checksum
89d9c30c1d48: Download complete
89d9c30c1d48: Pull complete
Digest: sha256:c19173c5ada610a5989151111163d28a67368362762534d8a8121ce95cf2bd5a
Status: Downloaded newer image for alpine:latest
Removing intermediate container 7fa2687f2385
 ---> 9c33554fd4cf
Successfully built 9c33554fd4cf
Successfully tagged nestybox/syscont-with-inner-containers:latest
```

We can see from above that the Docker build process has pulled the
busybox and alpine container images and stored them inside the system
container image. Cool!

Once the build is complete, we can optionally revert the
`default-runtime` config in the `/etc/docker/daemon.json` file we did
earlier (it's only needed for the Docker build, but not for running
the system container as we mentioned earlier).

Before proceeding, it's a good idea to prune any dangling images
created during the Docker build process to save storage.

```console
$ docker image prune
```

Now, let's run the newly created system container image:

```console
$ docker run --runtime=sysbox-runc -it --rm --hostname=syscont nestybox/syscont-with-inner-containers:latest
/ #
```

And let's start Docker inside the system container:

```console
/ # dockerd > /var/log/dockerd.log 2>&1 &

/ # docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
alpine              latest              965ea09ff2eb        2 days ago          5.55MB
busybox             latest              19485c79a9bb        7 weeks ago         1.22MB
```

As shown, the inner container images are already there. They came
pre-packaged with the system container!

This is cool because it allows us to create a pre-configured Docker
sandbox environment and capture them within a single, portable,
easy-to-deploy system container image.

In the next section we show how to do something similar but using
the `docker commit` command.

## Committing A System Container That Includes Inner Container Images

The `docker commit` command allows users to create a new container image
which contains a snapshot of the contents of a running container (excepting
bind mounts).

We can leverage this command to create a system container image that
includes images of inner containers.

We do this by deploying a system container, using Docker inside the
system container to pull inner container images, and then doing a
`docker commit` of the running system container into a new system
container image.

First, let's deploy a system container, start dockerd within it, and
pull some images inside:

```console
$ docker run --runtime=sysbox-runc -it --rm nestybox/alpine-docker

/ # dockerd > /var/log/dockerd.log 2>&1 &

/ # docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE

/ # docker pull busybox
Using default tag: latest
latest: Pulling from library/busybox
7c9d20b9b6cd: Pull complete
Digest: sha256:fe301db49df08c384001ed752dff6d52b4305a73a7f608f21528048e8a08b51e
Status: Downloaded newer image for busybox:latest

/ # docker pull alpine
Using default tag: latest
latest: Pulling from library/alpine
89d9c30c1d48: Pull complete
Digest: sha256:c19173c5ada610a5989151111163d28a67368362762534d8a8121ce95cf2bd5a
Status: Downloaded newer image for alpine:latest
```

Now, from the host, let's use Docker to "commit" the system container image (i.e.,
to take a snapshot of its contents):

```console
$ docker ps
CONTAINER ID        IMAGE                    COMMAND             CREATED             STATUS              PORTS               NAMES
31b9a7975749        nestybox/alpine-docker   "/bin/sh"           54 seconds ago      Up 52 seconds                           zen_mirzakhani

$ docker commit zen_mirzakhani nestybox/syscont-with-inner-containers:latest
sha256:82686f19cd10d2830e9104f46cbc8fc4a7d12c248f7757619513ca2982ae8464
```

A couple of restrictions apply here:

-   The `docker commit` instruction takes a `--pause` option which
    is set to `true` by default. Do not set it to `false` when trying to
    commit a system container image with inner containers. It won't work.

-   The `docker commit` instruction does not capture the contents of
    volumes or bind mounts mounted into the system container (per
    Docker's design).  This means that for the above example to work,
    we must not run the system container with a volume or bind mount
    into `/var/lib/docker` (such mounts are useful for persistence of
    inner container images, as described in this
    [section](#persistence-of-inner-container-images-with-docker-volumes)).

The commit operation may take anywhere from a few seconds to a few
minutes, depending on how many changes were done in the container's
files since it was created.

Let's run the newly committed system container image. If all is well, it
should contain the inner container images for busybox and alpine within it.

```console
$ docker run --runtime=sysbox-runc -it --rm nestybox/syscont-with-inner-containers:latest

/ # rm -f /var/run/docker.pid
/ # rm -f /run/docker/containerd/containerd.pid

/ # dockerd > /var/log/dockerd.log 2>&1 &

/ # docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
alpine              latest              965ea09ff2eb        3 days ago          5.55MB
busybox             latest              19485c79a9bb        7 weeks ago         1.22MB
```

There they are! This is cool because it gives us a mechanism to take a
snapshot of a running system container's contents that includes inner
container images. It's helpful as a way of saving work or exporting a
working system container for deployment in another machine (i.e.,
commit the image, docker push to a repo, and docker pull from another
machine).

One final note:

In the example above, we manually removed the `/var/run/docker.pid` and
`/run/docker/containerd/containerd.pid` files prior to starting the
Docker instance inside the committed system container. This was done
because the Docker commit captures the pid files of the Docker and
containerd instances running in the system container at the time the
commit occurred. If we don't remove these stale files, the Docker daemon
in the committed container may fail to start and report errors such
as:

```console
Error starting daemon: pid file found, ensure docker is not running or delete /var/run/docker.pid
```

or

```console
Failed to start containerd: timeout waiting for containerd to start
```

Such a failure does not occur when the system container has Systemd
inside, as the Systemd service scripts take care of ensuring the
Docker daemon starts correctly regardless of whether the docker.pid
file is present or not.

## Persistence of Inner Container Images with Docker Volumes

The Docker instance running inside a system container stores its
images in a cache located in the `/var/lib/docker` directory
inside the container.

When the system container is removed (i.e., not just stopped, but
actually removed via `docker rm`), the contents of that directory will
also be removed. In other words, inner Docker's image cache is
destroyed when the associated system container is removed.

It's possible to override this behavior by mounting host storage into
the system container's `/var/lib/docker` in order to persist the
inner Docker's image cache across system container life-cycles.

To do this, follow these steps:

1) Create a Docker volume on the host to serve as the persistent image cache for
   the Docker daemon inside the system container.

```console
$ docker volume create my-image-cache
my-image-cache

$ docker volume list
DRIVER              VOLUME NAME
local               my-image-cache
```

2) Launch the system container and mount the volume into the system
   container's `/var/lib/docker` directory.

```console
$ docker run --runtime=sysbox-runc -it --rm --hostname syscont --mount source=my-image-cache,target=/var/lib/docker nestybox/alpine-docker
/ #
```

3) Start Docker inside the system container:

```console
/ # dockerd > /var/log/dockerd.log 2>&1 &
```

4) Pull an inner container image (e.g. busybox):

```console
/ # docker pull busybox
Using default tag: latest
latest: Pulling from library/busybox
7c9d20b9b6cd: Pull complete
Digest: sha256:fe301db49df08c384001ed752dff6d52b4305a73a7f608f21528048e8a08b51e
Status: Downloaded newer image for busybox:latest

/ # docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             latest              19485c79a9bb        7 weeks ago         1.22MB
```

5) Exit the system container. Since it was started with the `--rm`
   option, Docker will remove the system container from the system.
   However, the contents of the system container's `/var/lib/docker`
   will persist since they are stored in volume `my-image-cache`.

6) Start a new system container and mount the `my-image-cache` volume:

```console
$ docker run --runtime=sysbox-runc -it --rm --hostname syscont --mount source=my-image-cache,target=/var/lib/docker nestybox/alpine-docker

/ # dockerd > /var/log/dockerd.log 2>&1 &

/ # docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             latest              19485c79a9bb        7 weeks ago         1.22MB
```

As shown, the inner container images persist across the life-cycle of
the system container. This is cool because it means that a system
container can leverage an existing Docker image cache stored somewhere
on the host, and thus avoid having to pull inner Docker images from
the network each time a new system container is started.

There are a couple of caveats to keep in mind:

* A persistent Docker image cache must only be mounted on a **single
  system container at any given time**. This is a restriction imposed
  by the Docker daemon, which does not allow its image cache to be
  shared concurrently among multiple daemon instances.  Sysbox will
  check for violations of this rule and report an appropriate error
  during system container creation.

* A persistent Docker image cache mounted into the system container's
  `/var/lib/docker` directory will "mask" any files present in that
  same directory as part of the system container's image. Such files
  would be present when using the system container build or commit
  features described [here](#building-a-system-container-that-includes-inner-container-images)
  and [here](#committing-a-system-container-that-includes-inner-container-images).

## Persistence of Inner Container Images with Bind Mounts

This section is similar to the prior one, but uses bind mounts instead
of Docker volumes when launching the system container.

The steps to do this are the following:

1) Create a directory on the host to serve as the persistent image cache for
   the Docker daemon inside the system container.

   As described in the [Sysbox User's Guide](usage.md#bind-mounts-in-exclusive-userns-remap-mode),
   the directory should be owned by a user in the range [0:65536] and
   will show up with those same user-IDs within the system
   container. In this example we choose user-ID 0 (root) so that the
   Docker instance inside the system container will see it's
   `/var/lib/docker` directory owned by `root:root` inside the system
   container.

   For extra security, we will also set the permission to 0700 as
   recommended in the Sysbox User's Guide.

```console
$ sudo mkdir /home/someuser/image-cache
$ sudo chmod 700 /home/someuser/image-cache
```

2) Launch the system container and bind-mount the newly created
   directory into the system container's `/var/lib/docker` directory.

```console
$ docker run --runtime=sysbox-runc -it --rm --hostname syscont --mount type=bind,source=/home/someuser/image-cache,target=/var/lib/docker nestybox/alpine-docker
/ #
```

3) Start Docker inside the system container and pull an image (e.g., busybox):

```console
/ # dockerd > /var/log/dockerd.log 2>&1 &

/ # docker pull busybox
Using default tag: latest
latest: Pulling from library/busybox
7c9d20b9b6cd: Pull complete
Digest: sha256:fe301db49df08c384001ed752dff6d52b4305a73a7f608f21528048e8a08b51e
Status: Downloaded newer image for busybox:latest

/ # docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             latest              19485c79a9bb        7 weeks ago         1.22MB
```

4) Exit the system container.

5) Start a new system container and bind-mount the `my-image-cache`
   directory as before:

```console
$ docker run --runtime=sysbox-runc -it --rm --hostname syscont --mount type=bind,source=/home/someuser/image-cache,target=/var/lib/docker nestybox/alpine-docker
```

6) Start Docker inside the system container and verify that it sees
   the images from the bind-mounted cache:

```console
/ # dockerd > /var/log/dockerd.log 2>&1 &
/ # docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES

/ # docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
busybox             latest              19485c79a9bb        7 weeks ago         1.22MB
```

## Sharing Storage Among System Containers

It's easy to share storage among multiple system containers by simply
bind-mounting the shared storage into each system container.

However, it requires that the permissions on the bind mounted
shared storage be set appropriately, which depends on the system
container isolation mode as described in the
[Sysbox User's Guide](usage.md#system-container-bind-mount-requirements).

In this example we assume Sysbox operates in exclusive userns-remap
mode (it's default isolation mode).

1) Create the shared storage on the host. In this example we use
   a Docker volume.

```console
$ docker volume create shared-storage
shared-storage
```

2) Create a system container and mount the shared storage volume into it:

```console
$ docker run --runtime=sysbox-runc -it --rm --hostname syscont --mount source=shared-storage,target=/mnt/shared-storage alpine:latest
/ #
```

3) From the system container, add a shared file to the shared storage:

```console
/ # touch /mnt/shared-storage/shared-file
/ # ls -l /mnt/shared-storage/shared-file
-rw-r--r--    1 root     root             0 Oct 24 22:08 /mnt/shared-storage/shared-file
```

4) In another shell, create another system container and mount the shared storage volume into it:

```console
$ docker run --runtime=sysbox-runc -it --rm --hostname syscont2 --mount source=shared-storage,target=/mnt/shared-storage alpine:latest
/ #
```

5) Confirm that the second system container sees the shared file:

```console
/ # ls -l /mnt/shared-storage/shared-file
-rw-r--r--    1 root     root             0 Oct 24 22:08 /mnt/shared-storage/shared-file
```

Notice that both system containers see the shared file with
`root:root` permissions, even though each system container is using
the Linux user namespace with exclusive user-ID and group-ID mappings
for enhanced security.

From the first system container:

```console
/ # cat /proc/self/uid_map
    0  268600992      65536
```

From the second system container:

```console
/ # cat /proc/self/uid_map
    0  268666528      65536
```

The reason both system containers see the correct `root:root`
ownership on the shared storage is through the magic of the Ubuntu
shiftfs filesystem, which Sysbox mounts over the shared storage.

From the first system container:

```console
/ # mount | grep shared-storage
/var/lib/docker/volumes/shared-storage/_data on /mnt/shared-storage type shiftfs (rw,relatime)
```

Finally, in the example above we used a Docker volume as the shared storage. However,
we could also use an arbitrary host directory as the shared storage. We need simply
bind-mount it to the system containers, though we must follow the requirements
for bind-mounts described in the [Sysbox User's Guide](usage.md#system-container-bind-mount-requirements).

## System Container Isolation Features

In this section we will show the following system container isolation
features by way of example:

-   Linux user namespace

-   Exclusive user-ID and group-ID mappings per system container

-   Linux capabilities

We assume Sysbox is configured in exclusive userns-remap mode (it's
default operating mode).

First let's deploy a system container:

```console
$ docker run --runtime=sysbox-runc --rm -it --hostname syscont debian:latest
root@syscont:/#
```

Nestybox system containers always use the Linux user-namespace (in fact
they use all Linux namespaces) to provide strong isolation between the
system container and the host.

Let's verify this by comparing the namespaces between a process inside
the system container and a process on the host.

From the system container:

```console
root@syscont:/# ls -l /proc/self/ns/
total 0
lrwxrwxrwx 1 root root 0 Oct 23 22:06 cgroup -> 'cgroup:[4026532563]'
lrwxrwxrwx 1 root root 0 Oct 23 22:06 ipc -> 'ipc:[4026532506]'
lrwxrwxrwx 1 root root 0 Oct 23 22:06 mnt -> 'mnt:[4026532504]'
lrwxrwxrwx 1 root root 0 Oct 23 22:06 net -> 'net:[4026532509]'
lrwxrwxrwx 1 root root 0 Oct 23 22:06 pid -> 'pid:[4026532507]'
lrwxrwxrwx 1 root root 0 Oct 23 22:06 pid_for_children -> 'pid:[4026532507]'
lrwxrwxrwx 1 root root 0 Oct 23 22:06 user -> 'user:[4026532503]'
lrwxrwxrwx 1 root root 0 Oct 23 22:06 uts -> 'uts:[4026532505]'
```

Now from the host:

```console
ls -l /proc/self/ns
total 0
lrwxrwxrwx 1 chino chino 0 Oct 23 22:07 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 chino chino 0 Oct 23 22:07 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 chino chino 0 Oct 23 22:07 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 chino chino 0 Oct 23 22:07 net -> 'net:[4026531992]'
lrwxrwxrwx 1 chino chino 0 Oct 23 22:07 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 chino chino 0 Oct 23 22:07 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 chino chino 0 Oct 23 22:07 user -> 'user:[4026531837]'
lrwxrwxrwx 1 chino chino 0 Oct 23 22:07 uts -> 'uts:[4026531838]'
```

You can see the system container uses dedicated namespaces, including
the user and cgroup namespaces.  It has no namespaces in common with
the host, which gives it stronger isolation compared to regular Docker
containers.

In addition, by default Sysbox assigns each system container exclusive
user-ID and group-ID mappings for each system container. This further
isolates system containers from the host and from each other.

```console
root@syscont:/# cat /proc/self/uid_map
         0  268994208      65536
root@syscont:/# cat /proc/self/gid_map
         0  268994208      65536
```

You can see the system container's users in the range [0:65535] are
mapped to a range of users on the host chosen by Sysbox. In this
example they map to the host user-ID range [268994208 :
268994208+65535].

Now let's now deploy another system container and check it's user-ID
and group-ID map:

```console
$ docker run --runtime=sysbox-runc --rm -it --hostname syscont2 debian:latest

root@syscont2:/# cat /proc/self/uid_map
         0  269059744      65536
root@syscont2:/# cat /proc/self/gid_map
         0  269059744      65536
```

Notice how Sysbox chose different user-ID and group-ID mappings for
this new system container. This provides isolation from the host as
well as from other system containers. More info on this can be found
in the [design guide](design.md#exclusive-user-namespace-mappings).

Now, let's check the capabilities of a root processes inside the
system container:

```console
root@syscont:/# grep Cap /proc/self/status
CapInh: 0000003fffffffff
CapPrm: 0000003fffffffff
CapEff: 0000003fffffffff
CapBnd: 0000003fffffffff
CapAmb: 0000003fffffffff
```

As shown, a root process inside the system container has all
capabilities enabled. However, those capabilities only take effect
with respect to host resources assigned to the system container
(courtesy of the Linux user namespace). In fact, a system container
process has no capabilities outside of the Linux user-namespace
associated with the system container, providing further isolation from
the host.

Contrast this to a regular Docker container. A root process in such a
container has a reduced set of capabilities (typically `CapEff:
00000000a80425fb`) and does not use the Linux user namespace. This has
two drawbacks:

1) The container's root process is limited in what it can do within the container.

2) The container's root process has those same capabilities on the host, which
   poses a higher security risk should the process escape
   the container's chroot jail.

System containers overcome both of these drawbacks.

## Sysbox Uninstallation

Refer to the [Sysbox README file](../README.md) for the uninstallation
instructions.

## Further reading

Refer to the [Sysbox User's Guide](usage.md) for details on the Sysbox
features shown in the above examples.

Also, the [Nestybox blog site](https://blog.nestybox.com) has more examples
on how to use system containers.
