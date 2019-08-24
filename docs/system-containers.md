System Containers
=================

## What are they?

A system container is a Linux container designed to run not just
applications, but also low-level system software.

Such software does not usually run in application containers as it
requires low level access to kernel resources which application
containers do not expose sufficiently or with proper isolation.

What type of system software are we talking about? Software such as
systemd, display servers, and even Docker and Kubernetes (that's
right, system containers support running these **inside** the
container).

Some other characteristics of system containers are:

* Use of the Linux user namespace (root inside the system container
  has all capabilities within it, but none on the host).

* Partially virtualized procfs (makes the container more closely
  resemble a real host).

* Entrypoint into a system management daemon (optional).


As with all containers, a user gets to choose what software runs in
the container. It can range from single binary to a complete system
environment (e.g., systemd, multiple apps, ssh server, display server,
docker, etc).

## Nestybox System Containers

While we are not inventing system containers per-se, Nestybox system
containers are the first to support deployment with Docker.

This allows users to leverage the power of Docker to package and deploy
system containers, and removes the need to learn about other container
engines.

Also, Nestybox system containers offer stronger isolation than
regular Docker application containers, by virtue of using the Linux
user namespace and exclusive user-ID mappings. This gives them strong
container-to-host and container-to-container isolation.

Finally, we are committed to make our system containers as complete an
abstraction of a virtual host as possible, to enable them to run a
large variety of applications and system level software with strong
isolation from the underlying host.

Our goal is to allow you run any software inside the system container
just as you would on a physical host. Ideally there shouldn't be any
difference.

## Use cases

Use cases for system containers are many. A few examples are:

* They can be used as alternatives to VMs. See
  [here](#comparison-to-VMs) for more on this.

* They can be used in CI/CD frameworks where deploying a container
  inside another container is useful (i.e., docker-in-docker).

* They can be used to setup an entire Kubernetes cluster in a
  developer's test machine, without resorting to VMs.

* They can be used to partition a cloud VM instance into multiple
  virtual hosts.

## How far along are we?

It's early days for Nestybox, so our system containers support a
reduced set of features. See [here](../README.md#Features) for a
list.

As a result, only a few of the use cases listed above are possible
currently. But we are working diligently on adding more.
See [here](../README.md#Roadmap) for our roadmap.

If you wish to provide us feedback, please see [here](../README.md#Feedback).
It's much appreciated!

## Comparison to VMs

In some ways, a system container resembles a VM. In particular, both
be used to package and deploy a complete system environment.

But system containers offer a different set of features & tradeoffs
when compared to VMs. Below are the pros/cons:

System Container Pros:

* Light, fast, and much more efficient than VMs (allowing for improved
  performance and much higher density).

* Flexible: you can configure the system container with the amount of
  software that you want, from a single binary to a full blown system
  with systemd, multiple apps, ssh server, docker, etc.

* Portable: the system container can run on bare-metal or in a
  cloud VM, in a high-end server or low-end device. It's not tied to a
  given hypervisor but rather to Linux (which runs almost everywhere).

* Easy-of-use: the system container can be packaged & deployed with
  Docker or Kubernetes. It voids the need for VM configuration &
  maintenance tools.

System Container Cons:

* It offers reduced isolation compared to a VM (i.e., system
  containers share the Linux kernel while VMs share the hypervisor;
  the attack surface of the former is wider than that of the latter).
  Thus, VMs are better suited for multi-tenant clouds with strict
  security requirements.

* It does not (currently) package hardware dependencies.

* A Linux system container must be deployed on a Linux host. It's not
  possible to deploy it on another platform (e.g., a Windows host).


Nestybox believes that in many scenarios, the pros heavily outweight
the cons.
