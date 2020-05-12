# Sysbox User Guide: Concepts & Terminology

These document describes concepts and terminology used by the Sysbox container
runtime. We use these throughout our documents.

## Contents

## Low-level Container Runtime

The software that given the the container's configuration and root filesystem
(i.e., a directory that has the contents of the container) interacts with the
Linux kernel to create the container.

Sysbox and the [OCI runc](https://github.com/opencontainers/runc) are examples
of low-level container runtimes.

The entity that provides the container's configuration and root filesystem to
the low-level container runtime is typically (but not necessarily) a high-level
container runtime (e.g., Docker, containerd).

## High-level Container Runtime

The high-level container runtime manages the container's lifecycle, from image
transfer and storage to container execution (by interacting with the low-level
container runtime).

Examples are Docker, containerd, cri-o, etc.

The [OCI runtime spec](https://github.com/opencontainers/runtime-spec) describes
the interface between the high-level container runtime and the low-level
container runtime.

## System Container

A container that is capable of executing system-level software such as Docker,
Kubernetes, Systemd, etc., in addition to application level software, with
proper isolation (i.e., without privileged containers) and without using
complex container entrypoints.

A system container typically (but not necessarily) packages multiple
services in it and is used like a "virtual host" environment, in
many ways similar to a virtual machine (VM).

You can deploy application containers within the system container, just
as you would on a physical host or VM.

See https://blog.nestybox.com/2019/09/13/system-containers.html for more info.

Sysbox is a low-level container runtime capable of creating system containers.

## Inner and Outer Containers

When launching Docker inside a system container, terminology can
quickly get confusing due to container nesting.

To prevent confusion we refer to the containers as the "outer" and
"inner" containers.

-   The outer container is a system container, created at the host
    level; it's launched with Docker + Sysbox.

-   The inner container is an application container, created within the outer
    container (e.g., it's created by the Docker or Kubernetes instance running
    inside the system container).

## Docker-in-Docker (DinD)

DinD refers to deploying Docker (CLI + Daemon) inside Docker containers.

Sysbox supports DinD using well-isolated (unprivileged) containers and without
the need for complex Docker run commands or specialized images.

## Kubernetes-in-Docker (KinD)

KinD refers to deploying Kubernetes inside Docker containers.

Each Docker container acts as a K8s node (replacing a VM or physical host).

A K8s cluster is composed of one or more of these containers, connected via an
overlay network (e.g., Docker bridge).

Sysbox supports KinD using well-isolated (unprivileged) containers and without
the need for complex Docker run commands or specialized images.
