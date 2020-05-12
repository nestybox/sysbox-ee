# Sysbox User Guide: Kubernetes-in-Docker

## Contents

## Intro

Nestybox system containers have preliminary support running Kubernetes (K8s)
inside system containers **without using privileged containers or complex
images**. This is known as Kubernetes-in-Docker or "KinD".

Each system container acts as a K8s node (replacing a VM or physical host).
A K8s cluster is composed of one or more of these system containers, connected
via an overlay network (e.g., Docker bridge).

TODO: Image showing KinD + Docker + Sysbox + k8s cluster inside containers; Mark KinD as optional.

This is useful for:

* Quickly and efficiently deploying Kubernetes clusters on a host

  - A 15-node K8s cluster can be deployed in 2->3 minutes on a small laptop, with < 5GB overhead!).

* Testing and CI/CD.

  - Deploy containerized K8s clusters into your CI/CD pipeline.

* Deploying multiple well-isolated K8s clusters within a single host.

  - Run multiple K8s clusters on a single host, with strong isolation and
    without resorting to VMs.

## Why use Nestybox System Containers for K8s?

Unlike all other alternatives, deploying K8s inside Nestybox system containers
gives you:

* Simplified images (no need for complex container images or entrypoints).

* Speed (a 15 node cluster can be deployed in 2 minutes on a regular laptop).

* Efficiency (Sysbox has features to significantly reduce the storage overhead).

* Full control of the container images and deployment process.

* Strong isolation (K8s node containers are fully unprivileged).

* The ability to create K8s node images that come preloaded with inner pod images.

And also: the Sysbox runtime is not just a Kubernetes-in-Docker solution. It's a
"any software in containers" solution, meaning that you can use it to deploy
other apps or system level software in containers easily and securely.

The following sections describe how to deploy K8s in system containers and get
these benefits.

## Deploying a K8s Cluster in System Containers

There are two recommended ways to deploy a K8s cluster using Nestybox system
containers:

1) Using a slightly modified version of the K8s.io KinD tool.

  - It's easy: the KinD tool takes care of setting up the cluster.

  - However, the container images used for the K8s nodes must be those created
    by the K8s.io KinD project. These are specialized images which are complex
    and harder to customize.

2) Directly via `docker run --runtime=sysbox-runc ...` commands.

  - This is easy too, but you need to manually sequence the creation of
    the containers that make up the K8s control plane and worker nodes.

  - TODO: Point to our sample bash script.

  - The upside is that the container images are simpler, easily customizable,
    and you work directly with Docker which gives you more flexibility and
    control. Inside the containers, kubeadm does the heavy lifting of
    configuring K8s.

The sections below describe both of these methods.

TODO: show a diagram to clarify these methods.

## Deploying a K8s cluster with K8s.io KinD + Sysbox

The [K8s.io KinD](https://kind.sigs.k8s.io) project produces a CLI tool called
"KinD" that enables deployment of Kubernetes clusters inside Docker containers.

It's an excellent tool that allows deployment of K8s clusters easily and quickly,
without resorting to VMs. The tool takes care of sequencing the creation of the
K8s cluster and configuring the container-based K8s nodes.

TODO: image showing kind + docker + sysbox + k8s cluster

### K8s.io KinD Shortcomings

KinD has some important shortcomings however which are overcomed when paired
with Sysbox.

The KinD tool uses Docker to create the containers that act as K8s nodes.
Docker in turn uses the default OCI runc runtime to create these containers.

Unfortunately, the OCI runc is not capable of creating containers that can
easily run system-level software (such as K8s).

As a result, the K8s.io KinD tool has the following shortcomings:

1) It relies on heavily specialized container images for the K8s nodes (e.g.,
   images with a complex Dockerfile and custom entrypoint). These are hard to
   customize.

2) It relies on very unsecure privileged containers. These are risky to use
   because the root user in the K8s node containers has full privileges on your
   host.

3) It consumes ~1GB of host storage per K8s container node. Thus, a 15-node
   cluster eats up ~15GB.

### K8s.io KinD + Sysbox

Shortcomings (2) and (3) can be overcomed when pairing the KinD tool with
Sysbox.

That is, when using KinD + Sysbox, your K8s cluster will be well isolated
(won't use privileged containers) and will use host storage more efficiently
(the 15-node cluster 15GB overhead is reduced to ~3GB). In addition, you
get features such as the ability to easily preload inner pod images into
the KinD K8s node images.

On the other hand, shortcoming (1) is inherent to the KinD tool. To overcome it,
you must deploy the K8s cluster directly with Docker + Sysbox, as described
[here](#deploying-a-k8s-cluster-with-docker-+-sysbox). This way you full
control the Docker image and avoid complex configurations or entrypoints.

### Requirements

To use KinD with Sysbox, you currently need two things:

1) You must use a slightly modified version of KinD (we call it `kind-sysbox`).

2) You must use a slightly modified K8s node container image.

The slightly modified version of KinD (kind-sysbox) is needed because the KinD
tool does not currently support container runtimes like Sysbox.

Fortunately, the changes to KinD in order to support Sysbox are tiny: we simply
change the way KinD invokes Docker when deploying the containers that make up
the K8s cluster nodes.

For example, kind-sysbox invokes Docker by passing the `--runtime=sysbox-runc`
flag and removing the `--privileged` flag.

Nestybox has forked the K8s.io KinD Github repo into [here](https://github.com/nestybox/kind)
and made the required changes.

Similarly, the slightly modified K8s node container image is needed because the
OCI runc binary inside the K8s node has a bug that prevents it from creating
pods correctly when running inside an unprivileged container.

TODO: point to the Dockerfile for the K8s node container image.

We will be working with the KinD community and OCI runc community to remove both
of these requirements in the near future.

### Example

Using k8s.io KinD + Sysbox is very easy, as shown below.

In the folowing example, `kind-sysbox` is the (slightly) modified version of
KinD for Sysbox (see prior section). And `nestybox/kindest-node:v1.18.2` is
the (slightly) modified K8s node image.

```console
$ kind-sysbox create cluster --image=nestybox/kindest-node:v1.18.2

Creating cluster "kind" ...
✓ Ensuring node image (nestybox/kindestnode:v1.18.2) 🖼
✓ Preparing nodes 📦 📦 📦 📦 📦 📦 📦 📦 📦 📦
✓ Writing configuration 📜
✓ Starting control-plane 🕹️
✓ Installing CNI 🔌
✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```

This creates a single node K8s cluster, using a container deployed with Docker +
Sysbox. It only takes ~35 seconds on my (not too fancy) laptop.

```console
$ docker ps
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS              PORTS                       NAMES
bab54b0a1060        nestybox/kindestnode:v1.18.2   "/usr/local/bin/entr…"   2 minutes ago       Up 2 minutes        127.0.0.1:43681->6443/tcp   kind-control-plane
```

You can verify the container is in fact an unprivileged container (a key
differentiating feature of Sysbox):

```console
$ docker exec kind-control-plane cat /proc/self/uid_map
         0     362144      65536
```

This means the container is using the Linux user namespace, and root in the
container (user 0) maps to unprivileged user-ID "362144" on the host.

Sysbox assigns an exclusive user-ID range to each node of the K8s cluster.

You can also verify the cluster is running properly:

```console
$ kubectl cluster-info --context kind-kind
Kubernetes master is running at https://127.0.0.1:43681
KubeDNS is running at https://127.0.0.1:43681/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.


$ kubectl get nodes
NAME                 STATUS   ROLES    AGE     VERSION
kind-control-plane   Ready    master   7m34s   v1.18.2


$ kubectl get all --all-namespaces
NAMESPACE            NAME                                             READY   STATUS    RESTARTS   AGE
kube-system          pod/coredns-66bff467f8-knrcv                     1/1     Running   0          7m22s
kube-system          pod/coredns-66bff467f8-wdzfr                     1/1     Running   0          7m22s
kube-system          pod/etcd-kind-control-plane                      1/1     Running   0          7m29s
kube-system          pod/kindnet-2xsqg                                1/1     Running   0          7m22s
kube-system          pod/kube-apiserver-kind-control-plane            1/1     Running   0          7m29s
kube-system          pod/kube-controller-manager-kind-control-plane   1/1     Running   0          7m29s
kube-system          pod/kube-proxy-rjrkr                             1/1     Running   0          7m22s
kube-system          pod/kube-scheduler-kind-control-plane            1/1     Running   0          7m29s
local-path-storage   pod/local-path-provisioner-bd4bb6b75-h6gww       1/1     Running   0          7m22s

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  7m40s
kube-system   service/kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   7m36s

NAMESPACE     NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/kindnet      1         1         1       1            1           <none>                   7m34s
kube-system   daemonset.apps/kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   7m36s

NAMESPACE            NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
kube-system          deployment.apps/coredns                  2/2     2            2           7m36s
local-path-storage   deployment.apps/local-path-provisioner   1/1     1            1           7m34s

NAMESPACE            NAME                                               DESIRED   CURRENT   READY   AGE
kube-system          replicaset.apps/coredns-66bff467f8                 2         2         2       7m22s
local-path-storage   replicaset.apps/local-path-provisioner-bd4bb6b75   1         1         1       7m22s
```

You can then manage the cluster as usual with kubectl (e.g., deploy pods, services, etc.)

To bring down the cluster, use:

```console
$ kind-sysbox delete cluster

Deleting cluster "kind" ...
```

### Multi-node Cluster

Deploying a multi-node cluster with KinD + Sysbox is very simple too. Just create
a KinD configuration file describing the nodes in the cluster.

For example, this config file creates a 10-node cluster:

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

To deploy the cluster, point to the config file:

```console
$ kind-sysbox create cluster --image=nestybox/kindestnode:v1.18.2 --config=/path/to/config.yaml

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

Have a nice day!
```

If all is well, you should see all K8s nodes ready:

```console
$ kubectl cluster-info --context kind-kind
Kubernetes master is running at https://127.0.0.1:33795
KubeDNS is running at https://127.0.0.1:33795/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'


$ kubectl get nodes
NAME                 STATUS   ROLES    AGE     VERSION
kind-control-plane   Ready    master   3m42s   v1.18.2
kind-worker          Ready    <none>   2m57s   v1.18.2
kind-worker2         Ready    <none>   2m58s   v1.18.2
kind-worker3         Ready    <none>   2m56s   v1.18.2
kind-worker4         Ready    <none>   3m      v1.18.2
kind-worker5         Ready    <none>   3m1s    v1.18.2
kind-worker6         Ready    <none>   3m      v1.18.2
kind-worker7         Ready    <none>   3m      v1.18.2
kind-worker8         Ready    <none>   3m1s    v1.18.2
kind-worker9         Ready    <none>   2m58s   v1.18.2
```

Once the cluster is ready you can manage the cluster as usual with kubectl.

To delete the multi-node cluster, use:

```console
$ kind-sysbox delete cluster
```

### Creating K8s node images that include inner pod images

Another key feature of Sysbox is that you can easily create K8s node images that
come preloaded with inner pod images.

There are two ways to do this:

* Via Docker build

* Via Docker commit

These are described below.

#### Building a K8s node image that includes inner pod images

This is done by using a Dockerfile that starts from the K8s node image, runs
the inner containerd, and commands containerd to pull the desired images.

But note: in order for this Dockerfile to work, Sysbox must be configured as
Docker's "default runtime".

This is needed because the Docker build process must call into Sysbox (otherwise
containerd won't start correctly inside the container) and unfortunately the
`docker build` command does not currently take a `--runtime` option (unlike
`docker run`).

To reconfigure the Docker daemon, we edit its `/etc/docker/daemon.json` file.
Here is how the `/etc/docker/daemon.json` file should look like. Notice the line
at the end of the file:

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
# systemctl restart docker.service
```

Once we've configured sysbox-runc as Docker's default runtime, we
can now build the system container image as follows:

1) Create a Dockerfile such as this one:

```Dockerfile
FROM nestybox/kindestnode:v1.18.2

# Pull inner images using containerd
COPY inner-image-pull.sh /usr/bin/
RUN chmod +x /usr/bin/inner-image-pull.sh && inner-image-pull.sh && rm /usr/bin/inner-image-pull.sh
```

This Dockerfile inherits from the K8s node image and copies a script
called "inner-image-pull.sh" into it. This script starts containerd and
directs it to pull the inner images. The script is then removed (we
don't want it in the final resulting image).

Here is the script:

```bash
#!/bin/sh

# containerd start
containerd > /var/log/containerd.log 2>&1 &
sleep 2

# use containerd to pull inner images into the k8s.io namespace (used by KinD).
ctr --namespace=k8s.io image pull docker.io/library/nginx:latest
```

The script must be placed in the same directory as the Dockefile:

```console
$ ls -l
total 8.0K
-rw-rw-r-- 1 cesar cesar 207 May 16 11:34 Dockerfile
-rw-rw-r-- 1 cesar cesar 228 May 16 12:02 inner-image-pull.sh
```

2) Now simply use `docker build` as usual:

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

Also, you can now revert Docker's default runtime configuration. Setting the
default runtime to Sysbox is only required for the Docker build process as
described previously.

3) Deploy the K8s cluster using the newly created image:

```console
$ kind-sysbox create cluster --image kindestnode-with-inner-nginx
```

4) Verify the K8s cluster nodes have the inner nginx image in them:

```console
$ docker exec kind-control-plane ctr --namespace=k8s.io image ls | awk '/nginx/ {print $1}'
docker.io/library/nginx:latest
```

There it is!

This is cool because you've just used a simple Dockefile to preload inner pod images
into your K8s nodes.

When you deploy pods using the nginx image, the image won't need
to be downloaded from the network (unless you modify the K8s [image pull policy](https://kubernetes.io/docs/concepts/containers/images/)).

You can preload as many images as you want, but keep in mind that they will add
to the size of your K8s node image.

A couple of important things to keep in mind:

* The inner containerd must pull images into the "k8s.io" namespace, as that's
  the one used by KinD K8s clusters.

* The inner nginx image is preloaded into all K8s nodes of the cluster (since
  they all use the same container image). This is as it should be.

In the following section, we show a different way of accomplishing the same
thing.

#### Commiting a K8s node image that includes inner pod images

You can use `docker commit` to create a K8s node image that comes preloaded
with inner pod images.

1) Deploy the K8s node image with Docker + Sysbox:

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
$ kind-sysbox create cluster --image=kindestnode-with-inner-nginx:latest
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
directly with Docker (rather than KinD) in step (1), so as to avoid any KinD
runtime configuration at the time we do the Docker commit in step (3).

## Deploying a K8s cluster with Docker + Sysbox

It's also possible to deploy a K8s cluster directly with Docker + Sysbox (i.e.,
without using the K8s.io KinD tool).

Compared to using the [K8s.io KinD tool + Sysbox](#deploying-a-k8s-cluster-with-k8s.io-kind-+-sysbox)),
the advantage is that the container images are simpler and you have **full control**
over the creation and configuration of the cluster. The drawback is that you
need to manage the cluster creation sequence, but it's pretty easy as you'll see.

TODO: image here

Below is sample procedure to deploy a two-node K8s cluster. You can easily
extend it for larger clusters.

We will be using the image `nestybox/k8s-node:v1.18.2` for the K8s node
containers. It's a simple image: it includes systemd, containerd, the K8s `kubeadm`
tool, and inner pod images for the K8s control plane. More on this
image [here](#the-k8s-node-image).

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

3) Next, setup kubectl.

In this example the kubectl is running inside the K8s control-plane node. But you can easily set it up
on the host (outside the containers) as shown [here](#using-kubectl-from-your-host).

NOTE: the single quotes in the command below are important to prevent shell
expansion of `$HOME` before the command is passed to the k8s-master container.

```console
$ docker exec k8s-master sh -c 'mkdir -p $HOME/.kube'
$ docker exec k8s-master sh -c 'cp -i /etc/kubernetes/admin.conf $HOME/.kube/config'
$ docker exec k8s-master sh -c 'chown $(id -u):$(id -g) $HOME/.kube/config'
```

4) Now apply the CNI config. In this example we use [Flannel](https://github.com/coreos/flannel#flannel).

```console
$ docker exec k8s-master sh -c "kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml"

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
$ docker exec k8s-master kubectl get all --all-namespaces

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

5) Now start a K8s worker node by launching another system container with
   Docker + Sysbox (similar to step (1)).

```console
$ docker run --runtime=sysbox-runc -d --rm --name=k8s-worker --hostname=k8s-worker nestybox/k8s-node:v1.18.2
```

6) Obtain the command that allows the worker join the cluster.

```console
$ join_cmd=$(docker exec k8s-master sh -c "kubeadm token create --print-join-command 2> /dev/null")
```

7) Tell the worker node to join the cluster:

NOTE: this command takes ~15 seconds to complete on my laptop.

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
$ docker exec k8s-master kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
k8s-master   Ready    master   2m15s   v1.18.2
k8s-worker   Ready    <none>   27s     v1.18.2
```

NOTE: it may take several seconds for all nodes in the cluster to reach the
"Ready" status.

9) Now let's create a pod deployment:

```console
$ docker exec k8s-master kubectl create deployment nginx --image=nginx
deployment.apps/nginx created

$ docker exec k8s-master kubectl scale --replicas=4 deployment nginx
deployment.apps/nginx scaled

$ docker exec k8s-master kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
nginx-f89759699-bwsct   1/1     Running   0          15s
nginx-f89759699-kl6bm   1/1     Running   0          15s
nginx-f89759699-rd7xg   1/1     Running   0          15s
nginx-f89759699-s8nzz   1/1     Running   0          20s
```

Cool, the pods are running!

From here on you use the K8s cluster as usual.

10) Just for fun, you can double-check the K8s cluster is using unprivileged
    containers:

```console
$ docker exec k8s-master cat /proc/self/uid_map
0     165536      65536

$ docker exec k8s-worker cat /proc/self/uid_map
0     231072      65536
```

Yes, they are:

* In the `k8s-master`, the root user in the container is mapped to unprivileged host user-ID 165536.

* In the `k8s-worker`, the root user is mapped to host unprivileged user-ID 231072.

Sysbox assigns each container an exclusive range of Linux user-namespace user-ID mappings.

11) After you are done, bring down the cluster:

```console
$ docker stop -t0 k8s-master k8s-worker
```

As you can see, the procedure is pretty simple and can easily be extended for
larger clusters (simply add more master or worker nodes as needed).

In fact, it's analogous to the procedure you would use when installing K8s on a
cluster made up of physical hosts or VMs: install `kubeadm` on each K8s node and
use it to configure the control-plane and worker nodes.

The reason this works so simply: the Sysbox container runtime creates the
containers that act as "virtual hosts" capable of running Kubernetes, without
requiring complex Docker images or unsecure privileged containers.

All you needed was a bunch of `docker run` commands plus invoking `kubeadm`
inside the containers was all you needed. And because of this, you can easily
script it.

TODO: point to the Nestybox bash script here.

### K8s Cluster on User-Defined Bridge Networks

In the example above, we deployed the K8s cluster inside Docker containers
connected to Docker's "bridge" network (the default network for all Docker
containers).

However, Docker's default bridge network has some limitations as described in
[this Docker article](https://docs.docker.com/network/bridge/).

To overcome those and improve the isolation of you K8s cluster, you can deploy
the containers that make up the K8s cluster on a Docker "user-defined bridge
network".

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

### Using Kubectl from your Host

In the example above, kubectl was running inside the K8s master node.

As a result, all kubectl commands start with `docker exec k8s-master kubectl ...`.

A simpler way would be to run kubectl on your host and point it to your
containerized K8s cluster. That way you can type `kubectl get pods` instead of
`docker exec k8s-master kubectl get pods`.

It's very easy to do this:

1) Install kubectl on your host as described [here](https://kubernetes.io/docs/tasks/tools/install-kubectl)

2) When deploying the k8s master node, expose port 6443 (the port where the K8s
   api-server listens):

```
$ docker run --runtime=sysbox-runc -d --rm --name=k8s-master --hostname=k8s-master --expose 6443 nestybox/k8s-node:v1.18.2
```

3) Then start `kubeadm init` in the k8s master node:

```
$ docker exec k8s-master sh -c "kubeadm init --kubernetes-version=v1.18.2 --pod-network-cidr=10.244.0.0/16"

[init] Using Kubernetes version: v1.18.2
[preflight] Running pre-flight checks
W0519 01:29:51.035470    1095 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[preflight] Pulling images required for setting up a Kubernetes cluster

...

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.17.0.2:6443 --token ddgjsg.4aufu0ejwzg5u8x2 \
--discovery-token-ca-cert-hash sha256:e56d48bbe7d516ebaa5752bbce7de014c855f0179157b2be3ad989e54fcc094f
```

4) Copy the `/etc/kubernetes/admin.conf` file in the k8s-master container to your host:

```
$ mkdir -p $HOME/.kube
$ docker cp k8s-master:/etc/kubernetes/admin.conf $HOME/.kube/config
```

5) Verify `kubectl` works:

```
$ kubectl get all --all-namespaces
NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
kube-system   pod/coredns-66bff467f8-m2884             0/1     Pending   0          3m39s
kube-system   pod/coredns-66bff467f8-xrgd6             0/1     Pending   0          3m39s
kube-system   pod/etcd-k8s-master                      1/1     Running   0          3m49s
kube-system   pod/kube-apiserver-k8s-master            1/1     Running   0          3m49s
kube-system   pod/kube-controller-manager-k8s-master   1/1     Running   0          3m49s
kube-system   pod/kube-proxy-kzbth                     1/1     Running   0          3m39s
kube-system   pod/kube-scheduler-k8s-master            1/1     Running   0          3m49s

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  3m58s
kube-system   service/kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   3m56s

NAMESPACE     NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   3m56s

NAMESPACE     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/coredns   0/2     2            0           3m56s

NAMESPACE     NAME                                 DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/coredns-66bff467f8   2         2         0       3m39s
```

From here on you can use `kubectl` on your host to control your cluster.

NOTE: depending on your cluster security requirements, exposing port 6443 in the
k8s-master container may or may not be appropriate. If you don't expose it, you
can access the cluster via other means (e.g., `docker exec` or via `ssh`
(assuming you add an ssh server to the K8s node image)).)

### The K8s node image

In the above example, the K8s node containers use the `nestybox/k8s-node:v1.18.2` image.

It's a sample image that we provide for reference; feel free to customize it as you need.
The Dockerfile is [here](../../dockerfiles/k8s-node).

Note how the Dockerfile makes use of a key feature of Sysbox: the ability to easily
preload inner images into the container when building the image.

The Dockerfile preloads inner pod images for the K8s control plane. This
significantly speeds up deployment, since the `kubeadm` instance inside the K8s
node need not download those at runtime.

The method used by the Dockerfile to preload the inner pod images into the
container is simple: during the build process, it invokes the `kubeadm` inside
the K8s node container and commands it to pull the inner pod images. The result
is a K8s system container image that includes the inner pods images. See the
`kube-pull.sh` script in the directory where the Dockerfile is.

The only wrinkle is that you must configure Sysbox as the Docker default
container runtime when building the image:

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

# systemctl restart docker.service
```

You can use this same method to easily preload your own inner pod images. For
example, you could add `containerd pull docker.io/library/nginx:latest` to the
kube-pull.sh script to preload the nginx pod image.

## Performance & Efficiency

By virtue of using containers as K8s nodes, Sysbox enables you to deploy
a multi-node clusters very quickly and super efficiently.

For example, I can deploy a 10-node K8s cluster on my laptop machine in
about 2 minutes with only 2.5GB of storage overhead (!).

The following sub-sections show some data.

### Performance when using K8s.io KinD + Sysbox

The table below compares the latency and storage overhead for deploying a K8s
cluster using the K8s.io KinD tool with and without Sysbox.

Data is for a 10 node cluster, collected on a 4 CPU, 8GB RAM laptop.

| Resource              |       K8s.io KinD        |  K8s.io KinD + Sysbox   |
|-----------------------|--------------------------|-------------------------|
| Storage overhead      |        9.8 GB            |       2.5 GB            |
| Cluster setup time    |        2 min             |       2 min             |
| Cluster teardown time |        6 sec             |       25 sec            |


As shown, the storage overhead savings with Sysbox are significant (75%). This
is because Sysbox has features that maximize container image layer sharing
between the containers that make up the K8s cluster. Though not shown, the
savings grow larger as the cluster size grows.

Latency-wise, the cluster creation time is similar.

The cluster deletion time however is much higher with Sysbox. The reason is that
some of Sysbox features rely on data movement between the container's root
filesystem and Sysbox-managed directories on the host, and this data movement
occurs both at cluster creation and cluster deletion.

### Performance when using Docker + Sysbox

TODO:

- collect perf numbers with Docker + Sysbox script (10-node cluster)

- use the k8s image with Docker inside.

- talk about inner Docker image sharing



K8s.io Kind: 3min, 30secs
Nestybox: 4min, 3GB




## System requirements

* K8s storage needs (kubelet fails with insufficient storage)


## Preliminary Support & Known Limitations

Sysbox's support for running K8s is preliminary at this point. Many
K8s features work, some don't, and some we've not tried yet.

### Known to Work

* Cluster deployment (single master, multi-worker).

* Cluster on Docker's default bridge network.

* Cluster on Docker's user-defined bridge network.

* Deploying multiple K8s clusters on a single host (each on it's own Docker user-defined bridge network).

* Kubeadm

* Kubectl

* Helm

* K8s deployments, replicas, auto-scale, rolling updates, daemonSets, configMaps, secrets, etc.

* K8s CNIs: Flannel.

* K8s services (ClusterIP, NodePort).

* K8s service mesh (Istio).

* K8s ingress controller (Traefik).

* K8s volumes (emptyDir, hostPath, persistent).

* Kube-proxy (iptables mode only).

### Not yet supported

* Kube-proxy ipvs mode.

* NFS volumes


## Debug Tips


## TODO

* Cleanup of port exposure.

* Scenario: add or remove nodes on the fly.

* Re-write kind section:

  - soften stance on KinD; don't point out shortcomings, rather say that Sysbox enhances it.

  - KinD is a partner, not a competitor.

* Focus on there features

  - Full control of images, simple docker run commands, etc.

* Complete "TODOs"

* break this into multiple docs (too large)

* talk about untainting master node to allow pods to be scheduled there

  `kubectl taint nodes --all node-role.kubernetes.io/master-`

* What about the k8s dashboard? Can we show an example?
