System Containers
=================

## What are they?

A system container is a Linux container designed to run low-level system
software, not just applications.

Such software does not usually run in application containers as it
requires low level access to kernel resources which application
containers do not expose sufficiently or with proper isolation.

What type of system software are we talking about? Software such as
systemd, display servers, and even Docker and Kubernetes (that's
right, system containers support running these **inside** the
container).

## Nestybox System Containers

While we are not inventing system containers per-se, Nestybox system
containers are the first to support deployment with Docker (and soon
Kubernetes).

This allows users to leverage the power of Docker to package and deploy
system containers, and removes the need to learn new tools.

In addition, Nestybox system containers offer the following:

* Use of the Linux user namespace (root inside the system container
  has all capabilities within it, but none on the host).

* Partially virtualized procfs (makes the container more closely
  resemble a real host).

* Ability for users to configure the system container from a single
  binary to a full system environment (e.g., systemd, multiple apps,
  ssh server, display server, docker, etc).

* Optional entrypoint into a system management daemon (e.g., systemd).

* Stronger isolation than regular Docker application containers, by
  virtue of using the Linux user namespace and exclusive user-ID
  mappings. This gives them strong container-to-host and
  container-to-container isolation.

We are committed to make our system containers as complete an
abstraction of a virtual host as possible.

Our goal is to allow you run any software inside the system container
just as you would on a physical host or virtual machine. And for this
to work out-of-the-box and securely, without complex configurations or
hacks.

## Use cases

The use cases we envision for system containers are many. A few
examples are:

* They can be used in CI/CD pipelines where the need to deploy
  Docker inside a Docker container (aka Docker-in-Docker) often arises.

* They can be used to setup an entire Kubernetes cluster in a
  developer's test machine, without resorting to VMs. This requires
  running Kubernetes inside the system container (we are working on
  it).

* They can be used to encapsulate development environments (e.g., one
  system container could be the code-writing environment, the other
  could be a test environment, etc.) instead of resorting to VMs for
  the same purpose.

Moreover, system containers are extremely portable, just like
application containers. You can deploy them on bare-metal, in a VM, in
a cloud VM instance, or even on edge or IoT devices (due to their
light-weight).

## How far along are we?

It's early days for Nestybox, so our system containers support a
reduced set of features. See [here](../README.md#Features) for a
list.

As a result, only a few of the use cases listed above are possible
currently. But we are working diligently on adding more.
See [here](../README.md#Roadmap) for our roadmap.

If you wish to provide us feedback, please see [here](../README.md#Feedback).
It's much appreciated!
