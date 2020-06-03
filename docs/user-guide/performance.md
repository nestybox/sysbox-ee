# Sysbox User Guide: Performance & Efficiency

## Contents

## Intro

This document describes Sysbox performance and efficiency
and related features.

TODO:

* Inner Docker Image Sharing

* Others?

* Show data, comparisons with alternatives.

  - Kata

  - Etc.

## Inner Docker Image Sharing

One of the nice features of Sysbox is that it easy to
[preload inner container images into system container images](images.md#preloading-inner-container-images-into-a-system-container)).

One of the side-effects of preloading inner container images is that the system
container images can quickly grow in size (typically hundreds of MBs).

To make matters worse, when the system container is created using that image,
Sysbox is forced to allocate more storage on the host in order to bypass
limitations associated with overlayfs nesting (i.e., overlayfs is the filesystem
used by Docker to create the container's filesystem).

For example, if a system container image is preloaded with inner Docker images
totaling a size of 500MB, each system container instance would normally require
that Sysbox allocate 500MB of storage on the host. If you deploy 10 system
containers, the overhead is 5GB. If you deploy 100 system containers, it grows
to 50GB. And so on. You get the point: the overhead can quickly grow.

To mitigate this, Sysbox has a feature called "inner Docker image sharing" that
**significantly** reduces the storage overhead. This feature works by ensuring
that multiple system containers created from the same image share preloaded
inner Docker image layers using Copy-on-Write (COW).

Continuing with the prior example, this feature allows you to deploy any number
of system containers and still only use 500MB of storage overhead for the
preloaded inner images! In other words, the storage overhead for preloaded inner
Docker images goes from O(n) to O(1), where 'n' is the number of system
containers.

Inner Docker image sharing is one of the key features that make Sysbox the most
efficient container runtime for deploying Docker or Kubernetes inside
containers.

As another example, see this storage overhead [table](kind.md#performance--efficiency)
for running Kubernetes in Docker containers with and without Sysbox.

### Effects on Container Startup Time

Inner Docker image sharing also improves the system container startup time.

The reason is that this feature causes Sysbox to move less data around when the
system container starts.

However, this improvement does not take effect on the first system
container instance based off a given image, but only on subsequent system
container instances based off the same image.

For example, the `nestybox/k8s-node` image is a system container used for
deploying Kubernetes-in-Docker. This image is preloaded with inner Docker
containers totaling up to 765MB.

When deploying 4 system containers with this image, notice the latency:

```console

cesar@eoan:$ time docker run --runtime=sysbox-runc -d nestybox/k8s-node:v1.18.2
fea256c7dc7dc28e5e4b8bc3a7419888dea99c825b69502545485d76158a678b

real    0m5.858s
user    0m0.027s
sys     0m0.028s


cesar@eoan:$ time docker run --runtime=sysbox-runc -d nestybox/k8s-node:v1.18.2
fc62a96cbb372ee5d16b28b5688ab4d331391fe20baa6add9e8758f6962dde47

real    0m0.991s
user    0m0.038s
sys     0m0.018s


cesar@eoan:$ time docker run --runtime=sysbox-runc -d nestybox/k8s-node:v1.18.2
26d4c94a3398604ea3f473ab6868ad359ffa1e30c5db20cd98922b1cd1591e5c

real    0m1.061s
user    0m0.030s
sys     0m0.027s

cesar@eoan:$ time docker run --runtime=sysbox-runc -d nestybox/k8s-node:v1.18.2
6b886f1c491c6cd593ec46f4f6076126309993add7b6426547283c5c9728da9e

real    0m0.953s
user    0m0.034s
sys     0m0.030s
```

The first system container instance took 5.8 seconds to deploy, and the rest
took ~1 second.

The reason for this is that when creating the first instance, Sysbox had to move
data for the inner container images. For subsequent instances this movement was
not required due to the inner Docker image sharing feature.

### Limitations of Inner Docker Image Sharing

There are a few limitations for inner Docker image sharing:

-   The storage savings apply only for inner container images that are [preloaded into the system container](#preloading-inner-container-images-into-a-system-container).
    They do not apply for inner images downloaded into the system container at runtime.

-   The storage savings apply only when the inner container images are Docker
    images, and to a lesser extend when they are Containerd images. They do not
    apply when using other container managers inside the system container.

### Disabling Inner Docker Image Sharing

Inner Docker image sharing is enabled by default in Sysbox.  It's possible to
disable it by passing the `--inner-docker-image-sharing=false` flag to the
sysbox-mgr.

See the [User Guide Configuration doc](configuration.md) for further info
on how to do this.
