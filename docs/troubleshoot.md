Sysboxd Troubleshooting
========================

## Installation Problems

When installing the Sysboxd package with the `dpkg` command
(see the [Installation instructions](../README.md#installation)), the expected output is:

```
Selecting previously unselected package sysboxd.
(Reading database ... 150254 files and directories currently installed.)
Preparing to unpack .../sysboxd_0.0.1-0~ubuntu-bionic_amd64.deb ...
Unpacking sysboxd (1:0.0.1-0~ubuntu-bionic) ...
Setting up sysboxd (1:0.0.1-0~ubuntu-bionic) ...

Disruptive changes made to docker configuration. Restarting docker service...
Created symlink /etc/systemd/system/sysboxd.service.wants/sysbox-fs.service → /lib/systemd/system/sysbox-fs.service.
Created symlink /etc/systemd/system/sysboxd.service.wants/sysbox-mgr.service → /lib/systemd/system/sysbox-mgr.service.
Created symlink /etc/systemd/system/multi-user.target.wants/sysboxd.service → /lib/systemd/system/sysboxd.service.
```

In case an error is observed above as a consequence of a missing
software dependency, proceed to download and install the missing
package(s) as indicated below. Once this requirement is satisfied,
Sysboxd's installation process will be automatically re-launched to
conclude this task.

Missing dependency output:

```
...
dpkg: dependency problems prevent configuration of sysboxd:
sysboxd depends on jq; however:
Package jq is not installed.

dpkg: error processing package sysboxd (--install):
dependency problems - leaving unconfigured
Errors were encountered while processing:
sysboxd
```

Install missing package by fixing (-f) system's dependency structures.

```
$ sudo apt-get install -f -y
```

Verify that Sysboxd's systemd units have been properly installed, and
associated daemons are properly running:

```
$ systemctl list-units -t service --all | grep sysboxd
sysbox-fs.service                   loaded    active   running sysbox-fs component
sysbox-mgr.service                  loaded    active   running sysbox-mgr component
sysboxd.service                     loaded    active   exited  Sysboxd General Service
```

The sysboxd.service is ephemeral (it exits once it launches the other sysboxd services).

## Sysboxd Logs

### sysbox-mgr & sysbox-fs

The Sysboxd daemons (i.e. sysbox-fs and sysbox-mgr) will log
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

## Bind Mount Security Check Error

When creating a system container with a bind mount, Docker may report the following error:

```bash
$ docker run --runtime=sysbox-runc -it --mount type=bind,source=/some/path,target=/mnt/path debian:latest

docker: Error response from daemon: OCI runtime create failed: ... shiftfs mountpoint security check failed: path /some/path is not exclusively accessible to the root user or group ...
```

This means that the bind mount security check has failed. This occurs when running with Docker
userns remap disabled (the default configuration for Docker) and the source of the bind
mount does not meet Sysboxd's security requirements. Refer to [Docker Bind Mount Permissions](usage.md#docker-bind-mount-permissions)
for details on how to fix this.

## Bind Mount Permissions Error

When running a system container with a bind mount, you may see that
the files and directories associated with the mount have
`nobody:nouser` ownership when listed from within the container.

This typically occurs when the source of the bind mount is owned by a
user on the host that is different from the user on the host to which
the system container's root user maps. Recall that Sysboxd containers
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
the Nestybox Shiftfs module is required by Sysboxd but is not loaded
in the Linux kernel:

```bash
# docker run --runtime=sysbox-runc -it debian:latest
docker: Error response from daemon: OCI runtime create failed:  container requires uid shifting but error was found: nbox_shiftfs module is not loaded in the kernel
```

To solve this problem, load the Nestybox Shiftfs module as described [here](https://github.com/nestybox/nbox-shiftfs-external).

Note that normally the Sysboxd installer loads this module into the
kernel, so this error implies that either the installer did not
succeed or that the module was somehow unloaded since then.
