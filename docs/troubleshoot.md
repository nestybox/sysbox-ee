Sysbox Troubleshooting
========================

## Installation Problems

When installing the Sysbox package with the `dpkg` command
(see the [Installation instructions](../README.md#installation)), the expected output is:

```
Selecting previously unselected package sysbox.
(Reading database ... 150254 files and directories currently installed.)
Preparing to unpack .../sysbox_0.0.1-0~ubuntu-bionic_amd64.deb ...
Unpacking sysbox (1:0.0.1-0~ubuntu-bionic) ...
Setting up sysbox (1:0.0.1-0~ubuntu-bionic) ...

Disruptive changes made to docker configuration. Restarting docker service...
Created symlink /etc/systemd/system/sysbox.service.wants/sysbox-fs.service → /lib/systemd/system/sysbox-fs.service.
Created symlink /etc/systemd/system/sysbox.service.wants/sysbox-mgr.service → /lib/systemd/system/sysbox-mgr.service.
Created symlink /etc/systemd/system/multi-user.target.wants/sysbox.service → /lib/systemd/system/sysbox.service.
```

In case an error is observed above as a consequence of a missing
software dependency, proceed to download and install the missing
package(s) as indicated below. Once this requirement is satisfied,
Sysbox's installation process will be automatically re-launched to
conclude this task.

Missing dependency output:

```
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

```
$ sudo apt-get install -f -y
```

Verify that Sysbox's systemd units have been properly installed, and
associated daemons are properly running:

```
$ systemctl list-units -t service --all | grep sysbox
sysbox-fs.service                   loaded    active   running sysbox-fs component
sysbox-mgr.service                  loaded    active   running sysbox-mgr component
sysbox.service                     loaded    active   exited  Sysbox General Service
```

The sysbox.service is ephemeral (it exits once it launches the other sysbox services).

## Sysbox Logs

### sysbox-mgr & sysbox-fs

The Sysbox daemons (i.e. sysbox-fs and sysbox-mgr) will log
information related to their activities in the
`/var/log/sysbox-fs.log` and `/var/log/sysbox-mgr.log` files
respectively. These logs should be useful during troubleshooting
exercises.

### sysbox-runc

For sysbox-runc, logging is handled as follows:

* When running Docker + sysbox-runc, the sysbox-runc logs are actually stored in
  a containerd directory such as:

  `/run/containerd/io.containerd.runtime.v1.linux/moby/<container-id>/log.json`

  where `<container-id>` is the container ID returned by Docker.

* When running sysbox-runc directly, sysbox-runc will not produce any logs by default.
  Use the `sysbox-runc --log` option to change this.

## Docker reports "Unknown runtime" error

When creating a system container, Docker may report the following error:

```
$ docker run --runtime=sysbox-runc -it ubuntu:latest
docker: Error response from daemon: Unknown runtime specified sysbox-runc.
```

This indicates that the Docker daemon is not aware of the sysbox-runc
runtime. This is likely caused by a misconfiguration of the
`/etc/docker/daemon.json` file. That file should have an entry for
sysbox-runc as follows:

```
# cat /etc/docker/daemon.json
{
   "runtimes": {
        "sysbox-runc": {
            "path": "/usr/local/sbin/sysbox-runc"
        }
    }
}
```

When this file is changed, the Docker daemon needs to be restarted:

```
# systemctl restart docker.service
```

**Note:** The sysbox installer automatically configures the `/etc/docker/daemon.json`
file to add the `sysbox-runc` runtime to it, and restarts the Docker daemon.

## Bind Mount Security Check Error

When creating a system container with a bind mount, Docker may report the following error:

```bash
$ docker run --runtime=sysbox-runc -it --mount type=bind,source=/some/path,target=/mnt/path debian:latest

docker: Error response from daemon: OCI runtime create failed: ... shiftfs mountpoint security check failed: path /some/path is not exclusively accessible to the root user or group ...
```

This means that the bind mount security check has failed. This occurs when running with Docker
userns remap disabled (the default configuration for Docker) and the source of the bind
mount does not meet Sysbox's security requirements. Refer to [Docker Bind Mount Permissions](usage.md#docker-bind-mount-permissions)
for details on how to fix this.

## Bind Mount Permissions Error

When running a system container with a bind mount, you may see that
the files and directories associated with the mount have
`nobody:nouser` ownership when listed from within the container.

This typically occurs when the source of the bind mount is owned by a
user on the host that is different from the user on the host to which
the system container's root user maps. Recall that Sysbox containers
always use the Linux user namespace and thus map the root user in the
system container to a non-root user on the host.

If the system container was created via Docker with userns-remap
disabled (the default configuration of Docker), then make sure that
the bind mount source has `root:root` ownership on the host.

If the system container was created via Docker with userns-remap
enabled, then make sure that the bind mount source has the same owner
(user:group) as the Docker userns remap configuration.

Refer to [Docker Bind Mount Permissions](usage.md#docker-bind-mount-permissions) for further
details.

## Nestybox Shiftfs Module Error

When creating a system container, the following error indicates that
the Nestybox Shiftfs module is required by Sysbox but is not loaded
in the Linux kernel:

```bash
# docker run --runtime=sysbox-runc -it debian:latest
docker: Error response from daemon: OCI runtime create failed:  container requires uid shifting but error was found: nbox_shiftfs module is not loaded in the kernel
```

To solve this problem, load the Nestybox Shiftfs module as described [here](https://github.com/nestybox/nbox-shiftfs-external).

Note that normally the Sysbox installer loads this module into the
kernel, so this error implies that either the installer did not
succeed or that the module was somehow unloaded since then.

## Unprivileged User Namespace Creation Error

When creating a system container, Docker may report the following error:

```bash
docker run --runtime=sysbox-runc -it ubuntu:latest
docker: Error response from daemon: OCI runtime create failed: host is not configured properly: kernel is not configured to allow unprivileged users to create namespaces: /proc/sys/kernel/unprivileged_userns_clone: want 1, have 0: unknown.
```

This means that the host's kernel is not configured to allow unprivileged users
to create user namespaces.

For Ubuntu, fix this with:

```
sudo sh -c "echo 1 > /proc/sys/kernel/unprivileged_userns_clone"
```

**Note:** The Sysbox package installer automatically executes this
instruction, so normally there is no need to do this configuration
manually.

## Failed to Setup Docker Volume Manager Error

When creating a system container, Docker may report the following error:

```bash
docker run --runtime=sysbox-runc -it ubuntu:latest
docker: Error response from daemon: OCI runtime create failed: failed to setup docker volume manager: host dir for docker store /var/lib/sysbox/docker can't be on ..."
```

This means that directory `/var/lib/sysbox` is on a filesystem not supported by Sysbox.

This directory must be on one of the following filesystems:

   * ext4
   * btrfs

The same requirement applies to the `/var/lib/docker` directory.

This is normally the case for vanilla Ubuntu installations.
