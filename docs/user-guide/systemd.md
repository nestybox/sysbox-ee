# Sysbox User Guide: Systemd-in-Docker

## Contents

## Intro

System containers can act as virtual host environments running multiple
services.

As such, running a process manager such as Systemd is useful (e.g., to start and
stop services in the appropriate sequence, perform zombie process reaping, etc.)

Moreover, many applications rely on Systemd in order to function properly (in
particular legacy (non-cloud) applications, but also cloud-native software such
as Kubernetes). If you want to run these in a container, Systemd must be present
in the container.

## Systemd-in-Docker

Nestybox has preliminary support for running Systemd inside a system
container, meaning that Systemd works but there are still some minor
issues that need resolution.

Deploying Systemd inside a system container is useful when you plan to
run multiple services inside the system container, or when you want to
use it as a virtual host environment.

Unlike other solutions, Nestybox system containers run Systemd securely and
easily, without resorting to privileged Docker containers and without the need
to create complex Docker run commands or specialized image entrypoints.

Simply launch a system container image that has Systemd as its entry point and
Sysbox will ensure the system container is setup to run Systemd without
problems.

The [Nestybox Dockerhub repo](https://hub.docker.com/u/nestybox) has a number of system container images that come
with systemd inside. The Dockerfiles for them are [here](../../dockerfiles).

The Sysbox Quick Start Guide has a [few examples](quickstart.md#deploy-a-system-container-with-systemd-inside) on
how to use them.

Of course, the container image will also need to have the systemd service units
that you need. These service units are typically added to the image during
the image build process. For example, the [Dockerfile](../../dockerfiles/ubuntu-bionic-systemd-docker/Dockerfile) for the
`nestybox/ubuntu-bionic-systemd-docker` image includes Docker's systemd service unit by simply
installing Docker in the container. As a result, when you launch that container, Systemd automatically
starts Docker.

## Systemd Alternatives

Systemd is great but may be a bit too heavy for your use case.

In that case you can use lighter-weight process managers such as
Supervisord.

The [Nestybox Dockerhub repo](https://hub.docker.com/u/nestybox) has a number of system container images that come
with Supervisord inside. The Dockerfiles for them are [here](../../dockerfiles).

The Sysbox Quick Start Guide has a [few examples](#deploy-a-system-container-with-supervisord-and-docker-inside).