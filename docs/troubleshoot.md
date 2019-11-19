# Sysbox Troubleshooting

## Contents

-   [Upgrading the Ubuntu Kernel](#upgrading-the-ubuntu-kernel)
    -   [Bionic Beaver](#bionic-beaver)
    -   [Disco Dingo](#disco-dingo)
-   [Sysbox Installation Problems](#sysbox-installation-problems)
-   [Ubuntu Shiftfs Module Not Present](#ubuntu-shiftfs-module-not-present)
-   [Unprivileged User Namespace Creation Error](#unprivileged-user-namespace-creation-error)
-   [Docker reports Unknown Runtime error](#docker-reports-unknown-runtime-error)
-   [Bind Mount Permissions Error](#bind-mount-permissions-error)
-   [Failed to Setup Docker Volume Manager Error](#failed-to-setup-docker-volume-manager-error)
-   [Failed to stat mount source at /var/lib/sysboxfs](#failed-to-stat-mount-source-at-varlibsysboxfs)
-   [Failed to register with sysbox-mgr](#failed-to-register-with-sysbox-mgr)
-   [Sysbox Logs](#sysbox-logs)
    -   [sysbox-mgr and sysbox-fs](#sysbox-mgr-and-sysbox-fs)
    -   [sysbox-runc](#sysbox-runc)

## Upgrading the Ubuntu Kernel

In case you need to upgrade your machine's Linux kernel to meet Sysbox [distro requirements](../README.md#supported-linux-distros),
here are some suggestions. Refer to the Ubuntu documentation online for further info.

### Bionic Beaver

For Bionic Beaver, we recommend using Ubuntu's [LTS-enablement](https://wiki.ubuntu.com/Kernel/LTSEnablementStack)
package:

```console
$ sudo apt-get update && sudo apt install --install-recommends linux-generic-hwe-18.04 -y
```

### Disco Dingo

For Disco Dingo, we recommend simply upgrading the distro:

```console
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt-get dist-upgrade
$ reboot
```

## Sysbox Installation Problems

When installing the Sysbox package with the `dpkg` command
(see the [Installation instructions](../README.md#installation)), the expected output is:

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
kernel, as newer Ubuntu kernels come with shiftfs.

The Ubuntu shiftfs module is required when Sysbox is configured in
[exclusive userns-remap mode](usage.md#exclusive-userns-remap-mode)
(it's default operating mode).

You can work-around this error by either:

-   Updating your Linux distro. See [here](../README.md#supported-linux-distros)
    for the list of Linux distros supported by Sysbox, and
    [here](#upgrading-the-ubuntu-kernel) for recommendations on how to
    update the distro.

or

-   Configuring Sysbox in docker userns-remap mode, as described
    [here](usage.md#system-container-isolation-modes). This
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

Refer to [System Container Bind Mount Requirements](usage.md#system-container-bind-mount-requirements) for
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

## Failed to stat mount source at /var/lib/sysboxfs

While creating a system container, Docker may report the following error:

```console
$ docker run --runtime=sysbox-runc -it alpine
docker: Error response from daemon: OCI runtime create failed: failed to create lib container mount: failed to stat mount source at /var/lib/sysboxfs/proc/sys: stat /var/lib/sysboxfs/proc/sys: no such file or directory: unknown.
```

This likely means that the sysbox-fs daemon is not running (for some reason).

Check if the sysbox-fs process is running via `ps`. Ideally it should look like this:

```console
$ ps -fu root | grep sysbox
root     23945     1  0 Nov12 pts/0    00:00:00 sysbox-fs --log-level=debug --log /dev/stdout
root     23946     1  0 Nov12 pts/0    00:00:00 sysbox-mgr --log-level=debug --log /dev/stdout
```

If sysbox-fs is missing from the `ps` output, stop and restart Sysbox via Systemd:

```console
$ sudo systemctl restart sysbox
```

## Failed to register with sysbox-mgr

While creating a system container, Docker may report the following error:

```console
$ docker run --runtime=sysbox-runc -it alpine
docker: Error response from daemon: OCI runtime create failed: failed to register with sysbox-mgr: failed to invoke Register via grpc: rpc error: code = Unavailable desc = all SubConns are in TransientFailure, latest connection error: connection error: desc = "transport: Error while dialing dial unix /run/sysbox/sysmgr.sock: connect: connection refused": unknown.
```

This likely means that the sysbox-mgr daemon is not running (for some reason).

Check if the sysbox-mgr process is running via `ps`. Ideally it should look like this:

```console
$ ps -fu root | grep sysbox
root     23945     1  0 Nov12 pts/0    00:00:00 sysbox-fs --log-level=debug --log /dev/stdout
root     23946     1  0 Nov12 pts/0    00:00:00 sysbox-mgr --log-level=debug --log /dev/stdout
```

If sysbox-mgr is missing from the `ps` output, stop and restart Sysbox via Systemd:

```console
$ sudo systemctl restart sysbox
```

## Sysbox Logs

### sysbox-mgr and sysbox-fs

The Sysbox daemons (i.e. sysbox-fs and sysbox-mgr) will log
information related to their activities in the
`/var/log/sysbox-fs.log` and `/var/log/sysbox-mgr.log` files
respectively. These logs should be useful during troubleshooting
exercises.

### sysbox-runc

For sysbox-runc, logging is handled as follows:

-   When running Docker + sysbox-runc, the sysbox-runc logs are actually stored in
    a containerd directory such as:

    `/run/containerd/io.containerd.runtime.v1.linux/moby/<container-id>/log.json`

    where `<container-id>` is the container ID returned by Docker.

-   When running sysbox-runc directly, sysbox-runc will not produce any logs by default.
    Use the `sysbox-runc --log` option to change this.
