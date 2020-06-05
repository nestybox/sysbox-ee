# Sysbox User Guide: Kubernetes-in-Docker

## Contents

-   [Intro](#intro)
-   [Use Cases](#use-cases)
-   [Deploying a K8s Cluster with Sysbox](#deploying-a-k8s-cluster-with-sysbox)
-   [Using K8s.io KinD + Sysbox](#using-k8sio-kind--sysbox)
-   [Using Kindbox](#using-kindbox)
-   [K8s Cluster Deployment with Docker + Sysbox](#k8s-cluster-deployment-with-docker--sysbox)
-   [The `nestybox/k8s-node` Image](#the-nestyboxk8s-node-image)
-   [Preloading Inner Pod Images into K8s Node Images](#preloading-inner-pod-images-into-k8s-node-images)
-   [Performance & Efficiency](#performance--efficiency)
-   [Preliminary Support & Known Limitations](#preliminary-support--known-limitations)

## Intro

Sysbox has preliminary support for running Kubernetes (K8s) inside system
containers. This is known as Kubernetes-in-Docker or "KinD".

TODO: Image showing Docker + Sysbox + k8s cluster inside containers

There are several [use-cases](#use-cases) for running Kubernetes-in-Docker.

While it's possible to run Kubernetes-in-Docker without Sysbox, doing so
requires complex Docker images, complex Docker run commands, and very unsecure
privileged containers.

Sysbox is the first container runtime capable of creating containers that can
run K8s seamlessly, using simple Docker images, no special configurations, and
strongly isolated containers (i.e,. using the Linux user-namespace).

You can deploy the cluster using simple Docker commands or using a higher
level tool (e.g., K8s.io KinD or Nestybox's "kindbox" tool).

With Sysbox, you have full control of the container images used for K8s
nodes. You can use different images for different cluster nodes if you wish, and
you can easily preload inner pod images into the K8s nodes.

Last but not least, Sysbox has features that significantly reduce the storage
overhead of the K8s cluster (e.g., from 10GB -> 3GB for a 10-node K8s cluster).
This enables you to deploy larger and/or more K8s clusters on a single host.

## Use Cases

Some sample use cases for Kubernetes-in-Docker are:

-   Deploy Kubernetes clusters quickly and efficiently:

    -   A 10-node K8s cluster can be deployed in under 2 minutes on a small laptop,
        with &lt; 1GB overhead.

-   Testing and CI/CD:

    -   Use it local testing or in a CI/CD pipeline.

-   Infrastructure-as-code:

    -   The K8s cluster is itself containerized, bringing the power of containers
        from applications down to infrastructure.

-   Increased host utilization:

    -   Run multiple K8s clusters on a single host, with strong isolation and
        without resorting to heavier VMs.

## Deploying a K8s Cluster with Sysbox

Deploying a K8s cluster is as simple as using Docker + Sysbox to deploy one or
more system containers, each with Systemd, Docker, and Kubeadm, and running
`kubeadm init` for the master node and `kubeadm join` on the worker node(s).

However, there are higher level tools that make it even easier. We support
two such tools currently:

-   [K8s.io KinD](https://kind.sigs.k8s.io) (slightly modified version)

-   Nestybox's `kindbox`.

Both of these are good choices, but there are some pros / cons between them.

K8s.io KinD offers a more standard way of doing the deployment, but relies on
complex container images and complex Docker commands, making it harder for you
to control the configuration and deployment of the cluster.

On the other hand, `kindbox` is a much simpler, open-source tool which Nestybox
provides as a reference example. It uses simple container images and Docker
commands, giving you **full control** over the configuration and deployment of
the cluster. Kindbox is also the the quickest and most efficient method to
deploy the K8s cluster with Sysbox.

The sections below we show how to deploy a K8s cluster with these tools, as well
as without them (through simple `docker run` commands).

## Using K8s.io KinD + Sysbox

The [K8s.io KinD](https://kind.sigs.k8s.io) project produces a CLI tool called
"kind" that enables deployment of Kubernetes clusters inside Docker containers.

It's an excellent tool that makes deployment of K8s cluster in containers fast &
easy. The tool takes care of sequencing the creation of the K8s cluster and
configuring the container-based K8s nodes.

When used with Sysbox, the capabilities of the K8s.io kind tool are enhanced:

-   The K8s cluster runs more efficiently (e.g., up to 70% less overhead per node).
    See [Performance & Efficiency](#performance--efficiency) for details.

-   The K8s cluster is properly isolated (no privileged containers!)

-   Sysbox makes it easy to preload the K8s node images with inner pod images
    (via a Dockerfile or Docker commit). This way, when you deploy the cluster,
    pod images of your choice are already embedded in the K8s nodes.

The figure below shows how K8s.io kind interacts with Sysbox.

TODO: image showing kind + docker + sysbox + k8s cluster

There is a wrinkle at this time however: the K8s.io kind tool does not yet
formally support the Sysbox runtime. Nestybox will be working with the
community to add this support.

In the meantime, we forked the K8s.io kind tool [here](https://github.com/nestybox/kind)
and made a few changes that enable it to work with Sysbox.

The changes are **very simple**: we just modify the way the K8s.io kind tool invokes
Docker to deploy the K8s node containers. For example, we add
`--runtime=sysbox-runc` to the Docker run command, and remove the `--privileged`
flag. The diffs are [here](https://github.com/nestybox/kind/commit/9708a130b7c0a539f2f3b5aa187137e71f747347).

### Modified K8s.io KinD Tool

Here is the process to download and build the forked version of the K8s.io kind
tool:

    $ git clone https://github.com/nestybox/kind.git
    $ cd kind
    $ make kind

The resulting binary is under `bin/kind`.

### Modified K8s node image

The K8s.io kind tool uses a custom Docker image for the K8s nodes called
`kindestnode/node`.

A slightly modified version of this image is needed for Sysbox.

-   NOTE: This should normally not be required, but unfortunately the `runc` binary inside
    the `kindest/node` image has a bug that prevents it from working properly in
    unprivileged containers in some cases. We will be working with the OCI runc
    community to resolve this bug.

The modified image is called `nestybox/kindestnode` and is uploaded in the Nestybox
Dockerhub repo [here](https://hub.docker.com/repository/docker/nestybox/kindestnode).
The Dockerfile is [here](../../dockerfiles/kindestnode).

### Deploying a cluster with K8s.io KinD + Sysbox

Once you've build the forked K8s.io kind tool, you use it as described
in the [kind user manual](https://kind.sigs.k8s.io/docs/user/quick-start/).

The only difference is that you need to explicitly tell the tool to use the
`nestybox/kindestnode` image.

When the kind tool deploys the K8s cluster, the fact that the K8s.io kind tool
is using Docker + Sysbox will not be obvious, except that you'll be able to
deploy K8s nodes with **much higher efficiency and security**.

For example, to deploy a 10-node K8s cluster, create a cluster config file:

```console
$ more config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
- role: worker
- role: worker
- role: worker
- role: worker
- role: worker
- role: worker
```

Then tell `kind` to create the cluster:

```console
$ kind create cluster --image=nestybox/kindestnode:v1.18.2 --config=config.yaml

Creating cluster "kind" ...
✓ Ensuring node image (nestybox/kindestnode:v1.18.2) 🖼
✓ Preparing nodes 📦 📦 📦 📦 📦 📦 📦 📦 📦 📦
✓ Writing configuration 📜
✓ Starting control-plane 🕹️
✓ Installing CNI 🔌
✓ Installing StorageClass 💾
✓ Joining worker nodes 🚜
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a nice day! 👋
```

It takes KinD + Sysbox less than 2 minutes and 3GB of storage to deploy that
10-node cluster. Not bad!

Without Sysbox, this same cluster eats up 10GB of storage. See section
[Performance & Efficiency](#performance--efficiency) below for more on this.

You can also verify the containers that make up the K8s cluster are in fact
unprivileged containers:

```console
$ docker exec kind-control-plane cat /proc/self/uid_map
         0     362144      65536
```

This means the container is using the Linux user namespace, and root in the
container (user 0) maps to unprivileged user-ID "362144" on the host.

Sysbox assigns an exclusive user-ID range to each node of the K8s cluster.

Next configure kubectl:

```console
$ kubectl cluster-info --context kind-kind
Kubernetes master is running at https://127.0.0.1:43681
KubeDNS is running at https://127.0.0.1:43681/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Then use it to verify K8s is running properly:

```console
$ kubectl get nodes
NAME                 STATUS   ROLES    AGE     VERSION
kind-control-plane   Ready    master   4m26s   v1.18.2
kind-worker          Ready    <none>   3m38s   v1.18.2
kind-worker2         Ready    <none>   3m34s   v1.18.2
kind-worker3         Ready    <none>   3m38s   v1.18.2
kind-worker4         Ready    <none>   3m34s   v1.18.2
kind-worker5         Ready    <none>   3m40s   v1.18.2
kind-worker6         Ready    <none>   3m35s   v1.18.2
kind-worker7         Ready    <none>   3m37s   v1.18.2
kind-worker8         Ready    <none>   3m39s   v1.18.2
kind-worker9         Ready    <none>   3m37s   v1.18.2
```

From here on, you can manage the cluster as usual with kubectl (e.g., deploy
pods, services, etc.)

Refer to the section [Preliminary Support & Known Limitations](#preliminary-support--known-limitations)
for a list of K8s functionality that works and doesn't when using Sysbox.

To bring down the cluster, use:

```console
$ kind delete cluster

Deleting cluster "kind" ...
```

The [K8s.io KinD website](https://kind.sigs.k8s.io/) for more info on how to use KinD.

## Using Kindbox

[Kindbox](../../scr/kindbox) is a simple, open-source tool created by Nestybox
to easily create K8s clusters with Docker + Sysbox.

It does some of the same things that the K8s.io KinD tool does (e.g., cluster
creation, destruction, etc.) but it's much simpler, does not require
specialized and complex container images, and it's even more efficient.

Kindbox also has a feature that allows you to resize the cluster dynamically,
and mix-and-match images for the K8s nodes.

Kindbox is a simple wrapper around Docker commands. It's open-source, so you
can copy it and modify it to your needs.

Kindbox is not meant to compete with the K8s.io KinD tool. Rather, it's meant to
provide a reference example of how easy it is to deploy a K8s cluster inside
containers when using the Sysbox container runtime.

By default, Kindbox uses a Docker image called `nestybox/k8s-node` for the K8s node
containers. It's a simple image that includes systemd, Docker, the K8s `kubeadm`
tool, and preloaded inner pod images for the K8s control plane. More on this
image [here](#the-nestyboxk8s-node-image).

Here is an example of how to use Kindbox:

```console
$ kindbox create --num-workers=9 mycluster

Creating a K8s cluster with Docker + Sysbox ...

Cluster name             : mycluster
Worker nodes             : 9
Docker network           : mycluster-net
Node image               : nestybox/k8s-node:v1.18.2
K8s version              : v1.18.2
Publish apiserver port   : false

Creating the K8s cluster nodes ...
  - Creating node mycluster-master
  - Creating node mycluster-worker-0
  - Creating node mycluster-worker-1
  - Creating node mycluster-worker-2
  - Creating node mycluster-worker-3
  - Creating node mycluster-worker-4
  - Creating node mycluster-worker-5
  - Creating node mycluster-worker-6
  - Creating node mycluster-worker-7
  - Creating node mycluster-worker-8

Initializing the K8s master node ...
  - Running kubeadm init on mycluster-master ... (may take up to a minute)
  - Setting up kubectl on mycluster-master ...
  - Initializing networking (flannel) on mycluster-master ...
  - Waiting for mycluster-master to be ready ...

Initializing the K8s worker nodes ...
  - Joining the worker nodes to the cluster ...

Cluster created successfully!

Use kubectl to control the cluster.

1) Install kubectl on your host
2) mkdir -p /home/cesar/.kube
3) docker cp mycluster-master:/etc/kubernetes/admin.conf /home/cesar/.kube/config
4) kubectl get nodes

Alternatively, use "docker exec" to control the cluster:

$ docker exec mycluster-master kubectl get nodes
```

This command creates a 10-node cluster called `mycluster` (one master node, 9
worker nodes).

It takes less than 2 minutes and consumes &lt; 1GB overhead on my laptop
machine!

In contrast, this same cluster requires 2.5GB when using K8s.io KinD +
Sysbox, and 10GB when using K8s.io KinD without Sysbox.

This means that with Sysbox, you can deploy large and/or more K8s clusters on
your machine quickly and without eating up your disk space.

The cluster nodes are on a Docker bridge network called `mycluster-net`. Kindbox
gives you full control of the Docker network to use for the cluster. Normally
each cluster would be on a dedicated network for extra isolation, but it's up to
you to decide. If you don't choose a network, Kindbox automatically creates one
for the cluster.

Let's setup kubectl on the host so we can control the cluster:

```console
$ docker cp mycluster-master:/etc/kubernetes/admin.conf /home/cesar/.kube/config
```

Now use kubectl to verify all is good:

```console
$ kubectl get nodes
NAME                 STATUS   ROLES    AGE     VERSION
mycluster-master     Ready    master   4m43s   v1.18.3
mycluster-worker-0   Ready    <none>   3m51s   v1.18.3
mycluster-worker-1   Ready    <none>   3m53s   v1.18.3
mycluster-worker-2   Ready    <none>   3m52s   v1.18.3
mycluster-worker-3   Ready    <none>   3m53s   v1.18.3
mycluster-worker-4   Ready    <none>   3m51s   v1.18.3
mycluster-worker-5   Ready    <none>   3m52s   v1.18.3
mycluster-worker-6   Ready    <none>   3m50s   v1.18.3
mycluster-worker-7   Ready    <none>   3m50s   v1.18.3
mycluster-worker-8   Ready    <none>   3m50s   v1.18.3
```

From here on, we use kubectl as usual to deploy pods, services, etc.

For example, to create an nginx deployment with 10 pods:

```console
$ kubectl create deployment nginx --image=nginx
$ kubectl scale --replicas=10 deployment nginx
$ kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP            NODE                 NOMINATED NODE   READINESS GATES
nginx-f89759699-6ch9m   1/1     Running   0          21s   10.244.11.4   mycluster-worker-6   <none>           <none>
nginx-f89759699-8jrc8   1/1     Running   0          21s   10.244.10.4   mycluster-worker-5   <none>           <none>
nginx-f89759699-dgxq8   1/1     Running   0          28s   10.244.2.15   mycluster-worker-1   <none>           <none>
nginx-f89759699-hx5tt   1/1     Running   0          21s   10.244.5.15   mycluster-worker-3   <none>           <none>
nginx-f89759699-l9v5p   1/1     Running   0          21s   10.244.1.10   mycluster-worker-0   <none>           <none>
nginx-f89759699-pdnhb   1/1     Running   0          21s   10.244.12.4   mycluster-worker-4   <none>           <none>
nginx-f89759699-qf46b   1/1     Running   0          21s   10.244.2.16   mycluster-worker-1   <none>           <none>
nginx-f89759699-vbnx5   1/1     Running   0          21s   10.244.3.14   mycluster-worker-2   <none>           <none>
nginx-f89759699-whgt7   1/1     Running   0          21s   10.244.13.4   mycluster-worker-8   <none>           <none>
nginx-f89759699-zblsb   1/1     Running   0          21s   10.244.14.4   mycluster-worker-7   <none>           <none>
```

Kindbox also allows you to easily resize the cluster (i.e., add or remove worker nodes).

For example, here we resize the cluster from 9 to 4 worker nodes:

```console
$ kindbox resize --num-workers=4 mycluster

Resizing the K8s cluster (current = 9, desired = 4) ...
  - Destroying node mycluster-worker-4
  - Destroying node mycluster-worker-5
  - Destroying node mycluster-worker-6
  - Destroying node mycluster-worker-7
  - Destroying node mycluster-worker-8
Done (5 nodes removed)
```

We can use kubectl to verify K8s no longer sees the removed nodes:

```console
$ kubectl get nodes
NAME                 STATUS   ROLES    AGE   VERSION
mycluster-master     Ready    master   32m   v1.18.3
mycluster-worker-0   Ready    <none>   31m   v1.18.3
mycluster-worker-1   Ready    <none>   31m   v1.18.3
mycluster-worker-2   Ready    <none>   31m   v1.18.3
mycluster-worker-3   Ready    <none>   31m   v1.18.3
```

Notice how K8s has re-scheduled the pods away to the remaining nodes:

```console
$ kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP            NODE                 NOMINATED NODE   READINESS GATES
nginx-f89759699-dgxq8   1/1     Running   0          10m   10.244.2.15   mycluster-worker-1   <none>           <none>
nginx-f89759699-hx5tt   1/1     Running   0          10m   10.244.5.15   mycluster-worker-3   <none>           <none>
nginx-f89759699-l6l7b   1/1     Running   0          28s   10.244.5.16   mycluster-worker-3   <none>           <none>
nginx-f89759699-l9v5p   1/1     Running   0          10m   10.244.1.10   mycluster-worker-0   <none>           <none>
nginx-f89759699-nbd2l   1/1     Running   0          28s   10.244.2.17   mycluster-worker-1   <none>           <none>
nginx-f89759699-qf46b   1/1     Running   0          10m   10.244.2.16   mycluster-worker-1   <none>           <none>
nginx-f89759699-rfklb   1/1     Running   0          28s   10.244.1.11   mycluster-worker-0   <none>           <none>
nginx-f89759699-tr9tr   1/1     Running   0          28s   10.244.1.12   mycluster-worker-0   <none>           <none>
nginx-f89759699-vbnx5   1/1     Running   0          10m   10.244.3.14   mycluster-worker-2   <none>           <none>
nginx-f89759699-xvx52   1/1     Running   0          28s   10.244.3.15   mycluster-worker-2   <none>           <none>
```

When resizing the cluster upwards, Kindbox allows you to choose the container
image for newly added K8s nodes. This means you can have a K8s cluster with a
mix of different node images. This is useful if you need some specialized K8s
nodes.

You can easily create multiple K8s clusters on the host by repeating the
`kindbox create` command. On my laptop (4 CPU & 8GB RAM), I am able to create
three small clusters without problem:

```console
$ kindbox list -l
NAME                   WORKERS         NET                  IMAGE                          K8S VERSION
cluster3               5               cluster3-net         nestybox/k8s-node:v1.18.2      v1.18.2
cluster2               5               cluster2-net         nestybox/k8s-node:v1.18.2      v1.18.2
mycluster              4               mycluster-net        nestybox/k8s-node:v1.18.2      v1.18.2
```

Moreover, the clusters are well isolated from each other: the K8s nodes are in
containers strongly secured via the Linux user namespace, and each cluster is in
a dedicated Docker network (for traffic isolation).

To destroy a cluster, simply type:

```console
$ kindbox destroy mycluster
Destroying K8s cluster "mycluster" ...
  - Destroying node mycluster-worker-0
  - Destroying node mycluster-worker-1
  - Destroying node mycluster-worker-2
  - Destroying node mycluster-worker-3
  - Destroying node mycluster-master
Done.
```

### Kindbox Simplicity

Kindbox is a very simple tool: it's a bash script wrapper around Docker commands
that create, destroy, and resize a Kubernetes-in-Docker cluster.

That is, Kindbox talks to Docker, Docker talks to Sysbox, and Sysbox creates or
destroys the containers.

The reason the tool is so simple: the Sysbox container runtime creates the
containers such that they can run K8s seamlessly inside. Thus, Kindbox
need only deploy the containers with Docker and run `kubeadm` within them to set
them up. **It's that easy**.

For this same reason, no specialized Docker images are needed for the containers
that act as K8s nodes. In other words, the K8s node image does not require
complex entrypoints or complex Docker commands for its deployment.

This is important because it enables you to fully control the contents of the
image and easily change it to your needs.

More on the image [here](#the-nestyboxk8s-node-image).

## K8s Cluster Deployment with Docker + Sysbox

It's also possible to deploy a K8s cluster directly with Docker + Sysbox,
without using the K8s.io `kind` or Nestybox's `kindbox` tools.

The drawback compared to using these tools is that you need to manage the K8s
cluster creation sequence. But it's pretty easy as you'll see.

Below is sample procedure to deploy a two-node K8s cluster. You can easily
extend it for larger clusters.

We will be using Docker image `nestybox/k8s-node:v1.18.2` for the K8s nodes
(same one we used for the `kindbox` example in the prior section). More on this
image [here](#the-nestyboxk8s-node-image).

Here is the sequence:

1) Deploy the K8s control plane node:

```console
$ docker run --runtime=sysbox-runc -d --rm --name=k8s-master --hostname=k8s-master nestybox/k8s-node:v1.18.2
```

2) Start `kubeadm` inside the K8s control plane node.

NOTE: this command takes ~30 seconds on my laptop.

```console
$ docker exec k8s-master sh -c "kubeadm init --kubernetes-version=v1.18.2 --pod-network-cidr=10.244.0.0/16"

W0516 21:49:30.490317    1817 validation.go:28] Cannot validate kube-proxy config - no validator is available
W0516 21:49:30.490343    1817 validation.go:28] Cannot validate kubelet config - no validator is available
[init] Using Kubernetes version: v1.17.2
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key

...

[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.17.0.2:6443 --token zvgtem.nt4vng7nodm1jwh9 \
--discovery-token-ca-cert-hash sha256:fc343a4e18923e65ad180414fb71627ae43b9d82292f12b566e9ba33686d3a43
```

3) Next, setup kubectl on the host.

Install kubectl on your host as described [here](https://kubernetes.io/docs/tasks/tools/install-kubectl)

Then copy the K8s `admin.conf` file from the container to the host.

```console
$ docker cp k8s-master:/etc/kubernetes/admin.conf $HOME/.kube/config
```

4) Now apply the CNI config. In this example we use [Flannel](https://github.com/coreos/flannel#flannel).

```console
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds-amd64 created
daemonset.apps/kube-flannel-ds-arm64 created
daemonset.apps/kube-flannel-ds-arm created
daemonset.apps/kube-flannel-ds-ppc64le created
daemonset.apps/kube-flannel-ds-s390x created
```

5) Verify all is good in the control-plane node:

```console
$ kubectl get all --all-namespaces

NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
kube-system   pod/coredns-6955765f44-czw7m             1/1     Running   0          9m14s
kube-system   pod/coredns-6955765f44-w9j2s             1/1     Running   0          9m14s
kube-system   pod/etcd-k8s-master                      1/1     Running   0          9m9s
kube-system   pod/kube-apiserver-k8s-master            1/1     Running   0          9m9s
kube-system   pod/kube-controller-manager-k8s-master   1/1     Running   0          9m9s
kube-system   pod/kube-flannel-ds-amd64-ps2z4          1/1     Running   0          60s
kube-system   pod/kube-proxy-fb2bd                     1/1     Running   0          9m14s
kube-system   pod/kube-scheduler-k8s-master            1/1     Running   0          9m8s

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  9m31s
kube-system   service/kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   9m30s

NAMESPACE     NAME                                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
kube-system   daemonset.apps/kube-flannel-ds-amd64     1         1         1       1            1           <none>                        60s
kube-system   daemonset.apps/kube-flannel-ds-arm       0         0         0       0            0           <none>                        60s
kube-system   daemonset.apps/kube-flannel-ds-arm64     0         0         0       0            0           <none>                        60s
kube-system   daemonset.apps/kube-flannel-ds-ppc64le   0         0         0       0            0           <none>                        60s
kube-system   daemonset.apps/kube-flannel-ds-s390x     0         0         0       0            0           <none>                        60s
kube-system   daemonset.apps/kube-proxy                1         1         1       1            1           beta.kubernetes.io/os=linux   9m30s

NAMESPACE     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/coredns   2/2     2            2           9m30s

NAMESPACE     NAME                                 DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/coredns-6955765f44   2         2         2       9m14s
```

NOTE: Wait for the `pod/coredns-*` pods to enter the "Running" state before
proceeding. It may take a few seconds.

5) Now start a K8s "worker node" by launching another system container with
   Docker + Sysbox (similar to step (1) above).

```console
$ docker run --runtime=sysbox-runc -d --rm --name=k8s-worker --hostname=k8s-worker nestybox/k8s-node:v1.18.2
```

6) Obtain the command that allows the worker join the cluster.

```console
$ join_cmd=$(docker exec k8s-master sh -c "kubeadm token create --print-join-command 2> /dev/null")

$ echo $join_cmd
kubeadm join 172.17.0.2:6443 --token 917kl0.xig3kbkv6ltc8qe6 --discovery-token-ca-cert-hash sha256:b64164e6ddc3ccbd65c71a04059f3db7ac4c57c9ad690621025b00f111883316
```

7) Tell the worker node to join the cluster:

NOTE: this command takes ~10 seconds to complete on my laptop.

```console
$ docker exec k8s-worker sh -c "$join_cmd"

W0516 22:07:52.240795    1366 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.17" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

Cool, the worker has joined.

8) Let's check that all looks good:

```console
$ docker ps
CONTAINER ID        IMAGE                       COMMAND             CREATED              STATUS              PORTS               NAMES
04aa10176cc1        nestybox/k8s-node:v1.18.2   "/sbin/init"        About a minute ago   Up About a minute   22/tcp              k8s-worker
f9e64edfc38e        nestybox/k8s-node:v1.18.2   "/sbin/init"        2 minutes ago        Up 2 minutes        22/tcp              k8s-master
```

```console
$ kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
k8s-master   Ready    master   2m15s   v1.18.2
k8s-worker   Ready    <none>   27s     v1.18.2
```

NOTE: it may take several seconds for all nodes in the cluster to reach the
"Ready" status.

9) Now let's create a pod deployment:

```console
$ kubectl create deployment nginx --image=nginx
deployment.apps/nginx created

$ kubectl scale --replicas=4 deployment nginx
deployment.apps/nginx scaled

$ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
nginx-f89759699-bwsct   1/1     Running   0          15s
nginx-f89759699-kl6bm   1/1     Running   0          15s
nginx-f89759699-rd7xg   1/1     Running   0          15s
nginx-f89759699-s8nzz   1/1     Running   0          20s
```

Cool, the pods are running!

From here on you use `kubectl` to manage the K8s cluster as usual (deploy pods, services, etc.)

10) Just for fun, you can double-check the K8s nodes are unprivileged
    containers:

```console
$ docker exec k8s-master cat /proc/self/uid_map
0     165536      65536

$ docker exec k8s-worker cat /proc/self/uid_map
0     231072      65536
```

Yes, they are:

-   In the `k8s-master`, the root user in the container is mapped to unprivileged host user-ID 165536.

-   In the `k8s-worker`, the root user is mapped to host unprivileged user-ID 231072.

Sysbox assigns each container an exclusive range of Linux user-namespace user-ID mappings.

11) After you are done, bring down the cluster:

```console
$ docker stop -t0 k8s-master k8s-worker
k8s-master
k8s-worker
```

As you can see, the procedure is pretty simple and can easily be extended for
larger clusters (simply add more master or worker nodes as needed).

All you needed was a bunch of `docker run` commands plus invoking `kubeadm`
inside the containers. In fact, the `kindbox` tool we described in the prior
section does basically this.

The procedure is analogous to the procedure you would use when installing K8s on a
cluster made up of physical hosts or VMs: install `kubeadm` on each K8s node and
use it to configure the control-plane and worker nodes.

The reason this works so simply: the Sysbox container runtime creates containers
that act as "virtual hosts" capable of running Kubernetes, without requiring
complex Docker run commands, container images, and unsecure privileged
containers.

### K8s Cluster on User-Defined Bridge Networks

In the example above, we deployed the K8s cluster inside Docker containers
connected to Docker's "bridge" network (the default network for all Docker
containers).

However, Docker's default bridge network has some limitations as described in
[this Docker article](https://docs.docker.com/network/bridge/).

To overcome those and improve the isolation of you K8s cluster, you can deploy
the K8s cluster nodes on a Docker "user-defined bridge network".

Doing so is trivial:

1) Create the user-defined bridge network for your cluster:

```console
$ docker network create mynet
629f7977fe9d8dbd10a4ca3ec99d68083f02371cca1623b255b03c52dc293782
```

2) Simply add the `--net=mynet` option when deploying the containers that make up the K8s cluster.

For example, deploy the K8s master with:

```console
$ docker run --runtime=sysbox-runc -d --rm --name=k8s-master --hostname=k8s-master --net=mynet nestybox/k8s-node:v1.18.2
```

Deploy the K8s worker nodes with:

```console
$ docker run --runtime=sysbox-runc -d --rm --name=k8s-worker-0 --hostname=k8s-worker-0 --net=mynet nestybox/k8s-node:v1.18.2
$ docker run --runtime=sysbox-runc -d --rm --name=k8s-worker-1 --hostname=k8s-worker-1 --net=mynet nestybox/k8s-node:v1.18.2
$ docker run --runtime=sysbox-runc -d --rm --name=k8s-worker-2 --hostname=k8s-worker-2 --net=mynet nestybox/k8s-node:v1.18.2
...
```

Follow the steps shown in the prior section for initializing the K8s master node
and joining the worker nodes to the cluster.

## The `nestybox/k8s-node` Image

In the above examples, the K8s node containers used the `nestybox/k8s-node` image.

It's a sample image that Nestybox provides for reference.

There is nothing special or complex about this image. It simply contains
systemd, Docker, kubeadm, and inner K8s control plane images [preloaded](#preloading-inner-pod-images-into-k8s-node-images).

Feel free to copy it and customize it per your needs. The Dockerfile is
[here](../../dockerfiles/k8s-node).

## Preloading Inner Pod Images into K8s Node Images

A key feature of Sysbox is that it allows you to easily create system container
images that come preloaded with inner container images.

You can use this to create K8s node images that come preloaded with inner pod
images.

This can significantly speed up deployment of the K8s cluster, since K8s node
need not download those inner pod images at runtime.

There are two ways to do this:

-   Using `docker build`

-   Using `docker commit`

Both of these are described in the sections below.

### Using Docker Build

Below is an example of how to use `docker build` to preload an inner pod image
into the `nestybox/kindestnode` image (i.e., the node image required for using
K8s.io KinD + Sysbox).

1) First, configure Docker's "default runtime" to `sysbox-runc`. This is only
   required during the image build process (i.e, you can revert the config
   once the build completes if you wish).

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

2) Stop all containers and restart the Docker service:

```console
$ docker stop $(docker ps -aq)
$ sudo systemctl restart docker.service
```

3) Create a Dockerfile such as this one:

```Dockerfile
FROM nestybox/kindestnode:v1.18.2

# Pull inner pod images using containerd
COPY inner-image-pull.sh /usr/bin/
RUN chmod +x /usr/bin/inner-image-pull.sh && inner-image-pull.sh && rm /usr/bin/inner-image-pull.sh
```

This Dockerfile inherits from the K8s node image and copies a script called
`inner-image-pull.sh` into it. Here is the script:

```bash
#!/bin/sh

# containerd start
containerd > /var/log/containerd.log 2>&1 &
sleep 2

# use containerd to pull inner images into the k8s.io namespace (used by KinD).
ctr --namespace=k8s.io image pull docker.io/library/nginx:latest
```

The script simply starts `containerd` and directs it to pull the inner
images. The script is then removed (we don't need it in the final resulting
image).

Note that the script must be placed in the same directory as the Dockerfile:

```console
$ ls -l
total 8.0K
-rw-rw-r-- 1 cesar cesar 207 May 16 11:34 Dockerfile
-rw-rw-r-- 1 cesar cesar 228 May 16 12:02 inner-image-pull.sh
```

4) Now simply use `docker build` as usual:

```console
$ docker build -t kindestnode-with-inner-nginx .

Sending build context to Docker daemon  3.072kB
Step 1/3 : FROM nestybox/kindestnode:v1.18.2
---> a1161162d7a5
Step 2/3 : COPY inner-image-pull.sh /usr/bin/
---> 1595afdbcc8f
Step 3/3 : RUN chmod +x /usr/bin/inner-image-pull.sh && inner-image-pull.sh && rm /usr/bin/inner-image-pull.sh
---> Running in d3a4910dfea8
docker.io/library/nginx:latest: resolving      |--------------------------------------|
elapsed: 0.1 s                  total:   0.0 B (0.0 B/s)
docker.io/library/nginx:latest: resolving      |--------------------------------------|
elapsed: 0.2 s                  total:   0.0 B (0.0 B/s)

...

docker.io/library/nginx:latest:                                                   resolved       |++++++++++++++++++++++++++++++++++++++|
index-sha256:30dfa439718a17baafefadf16c5e7c9d0a1cde97b4fd84f63b69e13513be7097:    done           |++++++++++++++++++++++++++++++++++++++|
manifest-sha256:8269a7352a7dad1f8b3dc83284f195bac72027dd50279422d363d49311ab7d9b: done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:11fa52a0fdc084d7fc3bbcb774389fd37b148ee98e7829cea4af189735acf848:    done           |++++++++++++++++++++++++++++++++++++++|
config-sha256:9beeba249f3ee158d3e495a6ac25c5667ae2de8a43ac2a8bfd2bf687a58c06c9:   done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:afb6ec6fdc1c3ba04f7a56db32c5ff5ff38962dc4cd0ffdef5beaa0ce2eb77e2:    done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:b90c53a0b69244e37b3f8672579fc3dec13293eeb574fa0fdddf02da1e192fd6:    done           |++++++++++++++++++++++++++++++++++++++|
elapsed: 4.7 s                                                                    total:  48.7 M (10.3 MiB/s)
unpacking linux/amd64 sha256:30dfa439718a17baafefadf16c5e7c9d0a1cde97b4fd84f63b69e13513be7097...
done
Removing intermediate container d3a4910dfea8
 ---> 698c1c7179a9
 Successfully built 698c1c7179a9
 Successfully tagged kindestnode-with-inner-nginx:latest
```

The output shows that during the Docker build, containerd run inside the
container and pulled the nginx image ... great!

Before proceeding, it's a good idea to prune any dangling images
created during the Docker build process to save storage.

```console
$ docker image prune
```

Also, you can now revert Docker's default runtime configuration (i.e. undo step
(1) above). Setting the default runtime to Sysbox is only required for the Docker
build process as described previously.

5) Deploy the K8s cluster using the newly created image:

```console
$ kind create cluster --image kindestnode-with-inner-nginx
```

6) Verify the K8s cluster nodes have the inner nginx image in them:

```console
$ docker exec kind-control-plane ctr --namespace=k8s.io image ls | awk '/nginx/ {print $1}'
docker.io/library/nginx:latest
```

There it is!

This is cool because you've just used a simple Dockerfile to preload inner pod images
into your K8s nodes.

When you deploy pods using the nginx image, the image won't need
to be pulled from the network (unless you modify the K8s [image pull policy](https://kubernetes.io/docs/concepts/containers/images/)).

You can preload as many images as you want, but note that they will add to the
size of your K8s node image.

A couple of important things to keep in mind:

-   In the example above we used containerd to pull the images, since that's
    the container manager inside the `nestybox/kindestnode` image.

-   The inner containerd must pull images into the "k8s.io" namespace since that's
    the one used by K8s inside the container.

-   If you wanted to preload inner pods into the `nestybox/k8s-node` image, the
    procedure is similar, except that the `inner-image-pull.sh` script would
    need to invoke Docker instead of containerd (since Docker is the container
    manager in that image).

    -   In fact, the [Dockerfile](../../dockerfiles/k8s-node) for the
        `nestybox/k8s-node` image does something even more clever: it invokes
        `kubeadm` inside the K8s node container to preload the inner images for the
        K8s control pods. Kubeadm in turn invokes the inner Docker, which pulls
        the pod images. The result is that the `nestybox/k8s-node` image has the
        K8s control-plane pods embedded in it, making deployment of the K8s
        cluster **much faster**.

### Using Docker Commit

You can also use `docker commit` to create a K8s node image that comes preloaded
with inner pod images.

Below is an example of how to do this with the `nestybox/kindestnode` image:

1) Deploy a system container using Docker + Sysbox:

```console
$ docker run --runtime=sysbox-runc --rm -d --name k8s-node nestybox/kindestnode:v1.18.2
```

2) Ask the containerd instance in the container to pull the desired inner images:

```console
$ docker exec k8s-node ctr --namespace=k8s.io image pull docker.io/library/nginx:latest
```

3) Take a snapshot of the container with Docker commit:

```console
$ docker commit k8s-node kindestnode-with-inner-nginx:latest
```

4) Stop the container:

```console
$ docker stop k8s-node
```

5) Launch the K8s cluster with KinD + Sysbox, using the newly committed image:

```console
$ kind create cluster --image=kindestnode-with-inner-nginx:latest
```

6) Verify the K8s cluster nodes have the inner nginx image in them:

```console
$ docker exec kind-control-plane ctr --namespace=k8s.io image ls | awk '/nginx/ {print $1}'
docker.io/library/nginx:latest
```

There it is!

This is cool because you've just used a simple Docker commit to preload inner
pod images into your K8s nodes.

One caveat however:

Doing a Docker commit of a running K8s node deployed via `kind create cluster`
won't work. The commit itself will complete, but the resulting image can't be
used in a new KinD cluster. The reason is that the committed image contains the
runtime configuration of the K8s node that is specific to the original cluster,
and that configuration likely won't work with new clusters.

The approach described above overcomes this by deploying the K8s node image
directly with Docker (rather than the `kind` tool) in step (1), so as to avoid
any K8s runtime configuration in the container at the time we do the Docker
commit in step (3).

## Performance & Efficiency

Sysbox enables you to deploy multi-node clusters quickly, efficiently, and
securely inside containers.

The table below compares the latency and storage overhead for deploying a K8s
cluster using the K8s.io `kind` tool (with and without Sysbox), as well as with
Nestybox's `kindbox` tool.

Data is for a 10 node cluster, collected on a small laptop with 4 CPUs and 8GB RAM.

| Metric                | K8s.io KinD | K8s.io KinD + Sysbox | Kindbox |
| --------------------- | ----------- | -------------------- | ------- |
| Storage overhead      | 10 GB       | 3 GB                 | 1 GB    |
| Cluster creation time | 2 min       | 2 min                | 2 min   |
| Cluster deletion time | 5 sec       | 20 sec               | 13 sec  |

As shown, the storage overhead reduction when using Sysbox is significant (> 70%).

This reduction is possible because Sysbox has features that maximize
[sharing of container image layers](images.md#inner-docker-image-sharing) between
the containers that make up the K8s cluster.

That sharing is even larger when Docker is running inside the container, which
is why the kindbox column shows the lowest overhead (by default kindbox uses the
`nestybox/k8s-node` image which comes with Docker inside).

Latency-wise, the cluster creation time is similar (~2 minutes for a 10 node
cluster, not bad!)

The cluster deletion time however is higher with Sysbox. The reason is that some
of Sysbox features rely on data movement between the container's root filesystem
and Sysbox-managed directories on the host, and this data movement occurs both
at cluster creation and cluster deletion.

## Preliminary Support & Known Limitations

Sysbox's support for KinD is preliminary at this stage.

This is because Kubernetes is a complex and large piece of software, and not all
K8s functionality works inside system containers yet.

However, many widely used K8s features work, so it's already quite useful.

Below is a list of K8s features that work and those that don't. Anything not
shown in the lists means we've not tested it yet (i.e., it may or may not work).

### Supported Functionality

-   Cluster deployment (single master, multi-worker).

-   Cluster on Docker's default bridge network.

-   Cluster on Docker's user-defined bridge network.

-   Deploying multiple K8s clusters on a single host (each on it's own Docker user-defined bridge network).

-   Kubeadm

-   Kubectl

-   Helm

-   K8s deployments, replicas, auto-scale, rolling updates, daemonSets, configMaps, secrets, etc.

-   K8s CNIs: Flannel.

-   K8s services (ClusterIP, NodePort).

-   K8s service mesh (Istio).

-   K8s ingress controller (Traefik).

-   K8s volumes (emptyDir, hostPath, persistent).

-   Kube-proxy (iptables mode only).

### Not yet supported

-   Kube-proxy ipvs mode.

-   K8s NFS volumes
