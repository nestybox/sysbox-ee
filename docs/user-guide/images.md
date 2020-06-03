# Sysbox User Guide: System Container Images

## Contents

-   [Intro](#intro)
-   [Nestybox Dockerhub Repo](#nestybox-dockerhub-repo)
-   [Preloading Inner Container Images into a System Container](#preloading-inner-container-images-into-a-system-container)
-   [Preloading Inner Container Image with Docker Build](#preloading-inner-container-image-with-docker-build)
-   [Preloading Inner Container Image with Docker Commit](#preloading-inner-container-image-with-docker-commit)
-   [Inner Docker Image Sharing](#inner-docker-image-sharing)

## Intro

This document describes aspects of System Container images.

The most important point: system container images are regular Docker
images. There is nothing special about them!

They usually carry system software in them (e.g., Systemd, Docker, K8s, etc.), but
can carry application software too.

You normally don't need any special or customized entrypoints in the system
container images. And you don't need complex Docker run commands to deploy them.

That's because the Sysbox runtime does the heavy lifting of ensuring that the
system software in the image runs correctly inside the container.

Since system container images often include Docker in them, it is useful
to preload inner container images inside the system container. Sysbox makes this
very easy. See [below](#preloading-inner-container-images-into-a-system-container) for info on
how to do this.

## Nestybox Dockerhub Repo

The [Nestybox Dockerhub repo](https://hub.docker.com/u/nestybox) has several images that
we provide as reference for users.

We often use these in the examples we provide in the User-Guide, Quickstart guide, and
Blog.

The Dockerfiles are [here](../../dockerfiles). Feel free to copy them and adapt them to
your needs.

If you see an error on them or think they can be improved, please file an
[issue](../issue-guidelines.md). Thanks!

## Preloading Inner Container Images into a System Container

Sysbox allows you to easily preload inner container images into a system container image.

This has several benefits:

-   Improves performance:

    -   If your system container is always deploying a common set of inner
        containers, you avoid the need for each system container to pull those from
        the network every time. Depending on the size of the inner image, it can
        significantly improve performance.

-   Improves efficiency:

    -   Sysbox has a [feature](performance.md#inner-docker-image-sharing) that maximizes sharing
        of preloaded inner container images across system containers. This
        **significantly** reduces the storage overhead on the host.

-   Ease of use:

    -   It's easier to deploy a system container that comes preloaded with your choice
        of inner containers, than to pull those inner containers into system container
        at runtime.

-   Air-gapped environments:

    -   In environments where there is no network connection, preloading the system
        container with inner container is a must have.

There are two ways preload inner container into a system container image:

-   Using `docker build`

-   Using `docker commit`

Both of these are described in the sections below.

## Preloading Inner Container Image with Docker Build

You can use a simple Dockerfile to preload inner container images into a system
container image.

Conceptually, the process is simple: the Dockefile for the system container
image has an instruction that requests the Docker instance inside the system
container to pull the inner container images. That's it.

There is a step-by-step example in the Sysbox [Quick-Start Guide](../quickstart.md#building-a-system-container-that-includes-inner-container-images).

The only wrinkle is that you must configure Sysbox as the Docker "default
runtime" when building the image (the example in the Quick-Start Guide shows
how to do this).

This is needed because during the build process, Docker is creating intermediate
containers for each Dockerfile instruction. Those intermediate containers must
be system containers deployed by Sysbox, since otherwise the inner Docker won't
run properly.

This process also works if the system container image has containerd inside
(rather than Docker). In this case, the Dockerfile must request containerd
to pull the inner images.

Nestybox uses this feature often. For example, the
[Dockerfile](../../dockerfiles/k8s-node) for our `nestybox/k8s-node` image (used
for running Kubernetes-in-Docker) preloads the Kubernetes pod images using this
same approach.

## Preloading Inner Container Image with Docker Commit

You can also use `docker commit` to preload inner container images into
a system container image.

The approach is also very simple: launch a system container that has an inner
Docker, use the inner Docker to pull the inner images, and then commit the
system container with the outer Docker. The committed image will include
the inner Docker images.

There is a step-by-step example in the [Quick-Start Guide](../quickstart.md#committing-a-system-container-that-includes-inner-container-images).

This approach is helpful as a way of saving work or exporting a working system
container for deployment in another machine (i.e., commit the system container
image, docker push to a repo, and docker pull from another machine).
