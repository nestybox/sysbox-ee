# Sysbox User Guide: Troubleshooting

## Contents

-   [Sysbox Installation Problems](#sysbox-installation-problems)
-   [Ubuntu Shiftfs Module Not Present](#ubuntu-shiftfs-module-not-present)
-   [Unprivileged User Namespace Creation Error](#unprivileged-user-namespace-creation-error)
-   [Docker reports Unknown Runtime error](#docker-reports-unknown-runtime-error)
-   [Bind Mount Permissions Error](#bind-mount-permissions-error)
-   [Failed to Setup Docker Volume Manager Error](#failed-to-setup-docker-volume-manager-error)
-   [Failed to register with sysbox-mgr or sysbox-fs](#failed-to-register-with-sysbox-mgr-or-sysbox-fs)
-   [Docker reports failure setting up ptmx](#docker-reports-failure-setting-up-ptmx)
-   [Docker exec fails](#docker-exec-fails)
-   [Sysbox Logs](#sysbox-logs)
-   [The /var/lib/sysbox is not empty even though there are no containers](#the-varlibsysbox-is-not-empty-even-though-there-are-no-containers)

## Sysbox Installation Problems

When installing the Sysbox package with the `dpkg` command
(see the [Installation instructions](../../README.md#installation)), the expected output is:

```console
Selecting previously unselected package sysbox.
(Reading database ... 150254 files and directories currently installed.)
Preparing to unpack .../sysbox_0.0.1-0~ubuntu-bionic_amd64.deb ...
Unpacking sysbox (1:0.0.1-0~ubuntu-bionic) ...
Setting up sysbox (1:0.0.1-0~ubuntu-bionic) ...

Non-disruptive changes made to docker configuration. Sending SIGHUP signal to docker daemon...

Created symlink /etc/systemd/system/sysbox.service.wants/sysbox-fs.service → /lib/systemd/system/sysbox-fs.service.
Created symlink /etc/systemd/system/sysbox.service.wants/sysbox-mgr.service → /lib/systemd/system/sysbox-mgr.service.
Created symlink /etc/systemd/system/multi-user.target.wants/sysbox.service → /lib/systemd/system/sysbox.service.
```

If your Docker daemon is configured with userns-remap enabled, you may also see the following:

```console
Disruptive changes made to docker configuration. Restarting docker service...
```

Both mean that the installation completed successfully.

In case an error occurs during installation as a consequence of a
missing software dependency, proceed to download and install the
missing package(s) as indicated below. Once this requirement is
satisfied, Sysbox's installation process will be automatically
re-launched to conclude this task.

Missing dependency output:

```console
...
dpkg: dependency problems prevent configuration of sysbox:
sysbox depends on jq; however:
Package jq is not installed.

dpkg: error processing package sysbox (--install):
dependency problems - leaving unconfigured
Errors were encountered while processing:
sysbox
```

Install missing package by fixing (-f) system's dependency structures.

```console
$ sudo apt-get update
$ sudo apt-get install -f -y
```

Verify that Sysbox's systemd units have been properly installed, and
associated daemons are properly running:

```console
$ systemctl list-units -t service --all | grep sysbox
sysbox-fs.service                   loaded    active   running sysbox-fs component
sysbox-mgr.service                  loaded    active   running sysbox-mgr component
sysbox.service                     loaded    active   exited  Sysbox General Service
```

The sysbox.service is ephemeral (it exits once it launches the other sysbox services),
so the `active exited` status above is expected.

## Ubuntu Shiftfs Module Not Present

When creating a system container, the following error indicates that
the Ubuntu shiftfs module is required by Sysbox but is not loaded
in the Linux kernel:

```console
# docker run --runtime=sysbox-runc -it debian:latest
docker: Error response from daemon: OCI runtime create failed: container requires user-ID shifting but error was found: shiftfs module is not loaded in the kernel. Update your kernel to include shiftfs module or enable Docker with userns-remap. Refer to the Sysbox troubleshooting guide for more info: unknown
```

The error likely means you are running Sysbox on an older Ubuntu
kernel, as newer Ubuntu kernels come with shiftfs. Or maybe you
are using a Ubuntu image meant for cloud VM instances, rather
than Ubuntu Desktop or Server.

You can work-around this error by either:

-   Updating your Linux distro. See the [distro compatibility](../distro-compat.md)
    document for the list of Linux distros supported by Sysbox, and
    recommendations on how to update the distro.

or

-   Configuring Docker in userns-remap mode, as described
    [here](../distro-compat.md#using-sysbox-on-kernels-without-the-shiftfs-module). This
    mode does not require use of shiftfs.

## Unprivileged User Namespace Creation Error

When creating a system container, Docker may report the following error:

```console
docker run --runtime=sysbox-runc -it ubuntu:latest
docker: Error response from daemon: OCI runtime create failed: host is not configured properly: kernel is not configured to allow unprivileged users to create namespaces: /proc/sys/kernel/unprivileged_userns_clone: want 1, have 0: unknown.
```

This means that the host's kernel is not configured to allow unprivileged users
to create user namespaces.

For Ubuntu, fix this with:

```console
sudo sh -c "echo 1 > /proc/sys/kernel/unprivileged_userns_clone"
```

**Note:** The Sysbox package installer automatically executes this
instruction, so normally there is no need to do this configuration
manually.

## Docker reports Unknown Runtime error

When creating a system container, Docker may report the following error:

```console
$ docker run --runtime=sysbox-runc -it ubuntu:latest
docker: Error response from daemon: Unknown runtime specified sysbox-runc.
```

This indicates that the Docker daemon is not aware of the sysbox-runc
runtime. This is likely caused by a misconfiguration of the
`/etc/docker/daemon.json` file. That file should have an entry for
sysbox-runc as follows:

```console
# cat /etc/docker/daemon.json
{
   "runtimes": {
        "sysbox-runc": {
            "path": "/usr/local/sbin/sysbox-runc"
        }
    }
}
```

If this file is changed, the Docker daemon needs to be restarted:

```console
# systemctl restart docker.service
```

**Note:** The sysbox installer automatically configures the `/etc/docker/daemon.json`
file to add the `sysbox-runc` runtime to it, and restarts the Docker daemon. Thus
this error is uncommon.

## Bind Mount Permissions Error

When running a system container with a bind mount, you may see that
the files and directories associated with the mount have
`nobody:nogroup` ownership when listed from within the container.

This typically occurs when the source of the bind mount is owned by a
user on the host that is different from the user on the host to which
the system container's root user maps. Recall that Sysbox containers
always use the Linux user namespace and thus map the root user in the
system container to a non-root user on the host.

Refer to [System Container Bind Mount Requirements](storage.md#system-container-bind-mount-requirements) for
info on how to set the correct permissions on the bind mount.

## Failed to Setup Docker Volume Manager Error

When creating a system container, Docker may report the following error:

```console
docker run --runtime=sysbox-runc -it ubuntu:latest
docker: Error response from daemon: OCI runtime create failed: failed to setup docker volume manager: host dir for docker store /var/lib/sysbox/docker can't be on ..."
```

This means that Sysbox's `/var/lib/sysbox` directory is on a
filesystem not supported by Sysbox.

This directory must be on one of the following filesystems:

-   ext4
-   btrfs

The same requirement applies to the `/var/lib/docker` directory.

This is normally the case for vanilla Ubuntu installations, so this
error is not common.

## Failed to register with sysbox-mgr or sysbox-fs

While creating a system container, Docker may report the following error:

```console
$ docker run --runtime=sysbox-runc -it alpine
docker: Error response from daemon: OCI runtime create failed: failed to register with sysbox-mgr: failed to invoke Register via grpc: rpc error: code = Unavailable desc = all SubConns are in TransientFailure, latest connection error: connection error: desc = "transport: Error while dialing dial unix /run/sysbox/sysmgr.sock: connect: connection refused": unknown.
```

or

```console
docker run --runtime=sysbox-runc -it alpine
docker: Error response from daemon: OCI runtime create failed: failed to pre-register with sysbox-fs: failed to register container with sysbox-fs: rpc error: code = Unavailable desc = all SubConns are in TransientFailure, latest connection error: connection error: desc = "transport: Error while dialing dial unix /run/sysbox/sysfs.sock: connect: connection refused": unknown.
```

This likely means that the sysbox-mgr and/or sysbox-fs daemons are not running (for some reason).

Check that these are running via systemd:

    $ systemctl status sysbox-mgr
    $ systemctl status sysbox-fs

If either of these services are not running, use Systemd to restart them:

```console
$ sudo systemctl restart sysbox
```

Normally Systemd ensures these services are running and restarts them automatically if
for some reason they stop.

## Docker reports failure setting up ptmx

When creating a system container with Docker + Sysbox, if Docker reports an error such as:

```console
docker: Error response from daemon: OCI runtime create failed: container_linux.go:364: starting container process caused "process_linux.go:533: container init caused \"rootfs_linux.go:67: setting up ptmx caused \\\"remove dev/ptmx: device or resource busy\\\"\"": unknown.
```

It likely means the system container was launched with the Docker `--privileged`
flag (and this flag is not compatible with Sysbox as described
[here](limitations.md)).

## Docker exec fails

You may hit this problem when doing an `docker exec -it my-syscont bash`:

    OCI runtime exec failed: exec failed: container_linux.go:364: starting container process caused "process_linux.go:94: executing setns process caused \"exit status 2\"": unknown

This occurs if the `/proc` mount inside the system container is set to "read-only".

For example, if you launched the system container and run the following command in it:

    $ mount -o remount,ro /proc

## Sysbox Logs

### sysbox-mgr and sysbox-fs

The Sysbox daemons (i.e. sysbox-fs and sysbox-mgr) will log information related
to their activities in `/var/log/sysbox-fs.log` and `/var/log/sysbox-mgr.log`
respectively. These logs should be useful during troubleshooting exercises.

You can modify the log level as described [here](configuration.md#reconfiguration-procedure).

### sysbox-runc

For sysbox-runc, logging is handled as follows:

-   When running Docker + sysbox-runc, the sysbox-runc logs are actually stored in
    a containerd directory such as:

    `/run/containerd/io.containerd.runtime.v1.linux/moby/<container-id>/log.json`

    where `<container-id>` is the container ID returned by Docker.

-   When running sysbox-runc directly, sysbox-runc will not produce any logs by default.
    Use the `sysbox-runc --log` option to change this.

## The /var/lib/sysbox is not empty even though there are no containers

Sysbox stores some container state under the `/var/lib/sysbox` directory
(which for security reasons is only accessible to the host's root user).

When no system containers are running, this directory should be clean and
look like this:

```console
# tree /var/lib/sysbox
/var/lib/sysbox
├── containerd
├── docker
│   ├── baseVol
│   ├── cowVol
│   └── imgVol
└── kubelet
```

When a system container is running, this directory holds state for the container:

```console
# tree -L 2 /var/lib/sysbox
/var/lib/sysbox
├── containerd
│   └── f29711b54e16ecc1a03cfabb16703565af56382c8f005f78e40d6e8b28b5d7d3
├── docker
│   ├── baseVol
│   │   └── f29711b54e16ecc1a03cfabb16703565af56382c8f005f78e40d6e8b28b5d7d3
│   ├── cowVol
│   └── imgVol
└── kubelet
    └── f29711b54e16ecc1a03cfabb16703565af56382c8f005f78e40d6e8b28b5d7d3
```

If the system container is stopped and removed, the directory goes back to it's clean state:

```console
# tree /var/lib/sysbox
/var/lib/sysbox
├── containerd
├── docker
│   ├── baseVol
│   ├── cowVol
│   └── imgVol
└── kubelet
```

If you have no system containers created yet `/var/lib/sysbox` is not clean, it means
Sysbox is in a bad state. This is very uncommon as Sysbox is well tested.

To overcome this, you'll need to follow this procedure:

1) Stop and remove all system containers (e.g., all Docker containers created with the sysbox-runc runtime).

-   There is a bash script to do this [here](../../scr/rm_all_syscont).

2) Restart Sysbox:

```console
$ sudo systemctl restart sysbox
```

3) Verify that `/var/lib/sysbox` is back to a clean state:

```console
# tree /var/lib/sysbox
/var/lib/sysbox
├── containerd
├── docker
│   ├── baseVol
│   ├── cowVol
│   └── imgVol
└── kubelet
```

## Kubernetes-in-Docker fails to create pods

When running [K8s-in-Docker](kind.md), if you see pods failing to deploy, we suggest starting
by inspecting the kubelet log inside the K8s node where the failure occurs.

```
$ docker exec -it <k8s-node> bash

# journalctl -u kubelet
```

This log often has useful information on why the failure occurred.

One common reason for failure is that the host is lacking sufficient storage. In
this case you'll see a message like this one in the kubelet log:

```
Disk usage on image filesystem is at 85% which is over the high threshold (85%). Trying to free 1284963532 bytes down to the low threshold (80%).
```

To overcome this, make some more storage room in your host and redeploy the
pods.
