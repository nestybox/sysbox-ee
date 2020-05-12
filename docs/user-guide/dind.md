# Sysbox User Guide: Docker-in Docker

## Contents


## Intro

Nestybox system containers support running Docker inside the system container
(aka Docker-in-Docker), **without using privileged containers, complex images,
or bind-mounting the host's Docker socket into the container**.

In other words, you can run Docker inside the container cleanly and securely,
with **total isolation** between the inner and outer Docker daemons.

This is useful for Docker sandboxing, testing, and CI/CD use cases.

And it's fast: Sysbox sets up the system container such that the Docker daemon
inside the container can use its fast overlay2 storage driver, rather than
alternative Docker-in-Docker solutions that resort to the slower vfs driver.

## Installing Docker inside the Container

To install Docker inside a system container, the easiest way is to use a system
container image that has Docker pre-installed in it.

You can find a few such images in the [Nestybox DockerHub repo](https://hub.docker.com/r/nestybox). The
Dockefiles for the images are [here](../../dockerfiles).

Alternatively, you can always deploy a baseline system container image (e.g.,
ubuntu or alpine) and install Docker in it just as you would on a physical host
or VM. In fact, the system container images that come with Docker preinstalled
are created with a Dockerfile that does just that.

## Running Docker inside the Container

You do this just as you would on a physical host or VM (e.g., `docker run -it alpine`).

The [Sysbox Quickstart Guide](../quickstart.md) has several examples showing how
to run Docker inside a system container.

## Pre-Installing Inner Container Images

In a system container image not only can you pre-install Docker, you can also
easily pre-install inner Docker container images.

This way, when you deploy the system container, you get Docker plus whatever
inner container images you desire. Yo avoid the need to download those inner
images from the network when the system container is running.

There are two ways to do this:

1) Via a simple Dockerfile: see the Quickstart guide [here](../quickstart.md#building-a-system-container-that-includes-inner-container-images) for an example.

2) Using Docker commit: see the Quickstart guide [here](../quickstart.md#committing-a-system-container-that-includes-inner-container-images) for an example.


Note that system container images that include inner container images can
quickly grow in size. This is turn causes a slight delay (few seconds) when
deploying the container.

## Persistence of Inner Docker Images

Inner container images that are preloaded within the system container image are
persistent: they are present every time a new system container is created.

However, inner container images that are downloaded into the system container at
runtime are not by default persistent: they get destroyed when the system
container is removed (not just stopped, but actually removed via `docker rm`).

But it's easy to make these runtime inner container images persistent too.

This is done by simply mounting a host directory into the system container's
`/var/lib/docker` directory (i.e., where the inner Docker stores its container
images).

The Sysbox Quick Start Guide has examples [here](../quickstart.md#persistence-of-inner-container-images-with-docker-volumes)
and [here](../quickstart.md#persistence-of-inner-container-images-with-bind-mounts).

A warning though:

    A given host directory mounted into a system container's `/var/lib/docker` must
    only be mounted on a **single system container at any given time**. This is a
    restriction imposed by the inner Docker daemon, which does not allow its image
    cache to be shared concurrently among multiple daemon instances. Sysbox will
    check for violations of this rule and report an appropriate error during system
    container creation.

## Inner Docker Privileged Containers

Inside a system container you *can* deploy privileged Docker containers (e.g.,
by issued the following command to the inner Docker: `docker run --privileged ...`).

The ability to run privileged containers inside a system container is useful
when deploying inner containers that require full privileges (typically
containers containing system services such as Kubernetes control-plane pods).

Note however that a privileged container inside a system container is privileged
within the context of the system container only, but has **no privileges on the
underlying host**.

For example, when running a privileged container inside a system container, the
`/proc` mounted inside the privileged container only allows access to resources
associated with the system container. It does **not** allow access to all host
resources.

This is a key security feature of Sysbox: it allows you to run privileged
containers inside a system container without risking host security.

## Limitations for the Inner Docker

This section describes limitations for the inner Docker.

### Inner Docker Data-Root

The inner Docker must store it's images at the usual `/var/lib/docker`. This
directory is known as the Docker "data-root".

While it's possible to configure the inner Docker to store it's images at some
other location within the system container (via the Docker daemon's
`--data-root` option), Sysbox does **not** currently support this (i.e., the
inner Docker won't work).

### Inner Docker userns-remap

The inner Docker must not be configured with [userns-remap](https://docs.docker.com/engine/security/userns-remap/).

Enabling userns-remap in Docker is sometimes done in order to improve isolation
of containers (by leveraging the isolation provided by the Linux user-namespace
on them).

In this case, enabling userns-remap on the inner Docker would cause the Linux
user-namespace to be used on the inner containers, further isolating them from
the rest of the software in the system container.

This is useful and we plan to support in the future.

But note the following: even without the inner Docker userns remap, inner
containers are already well isolated from the host by the system container
(which does use the Linux user namespace).