Sysvisor Troubleshooting
========================

## Installation Problems

When installing the Sysvisor package with the dpkg command
(see the [Installation instructions](../README.md#installation)), the expected output is:

```
Selecting previously unselected package sysvisor.
(Reading database ... 150254 files and directories currently installed.)
Preparing to unpack .../sysvisor_0.0.1-0~ubuntu-bionic_amd64.deb ...
Unpacking sysvisor (1:0.0.1-0~ubuntu-bionic) ...
Setting up sysvisor (1:0.0.1-0~ubuntu-bionic) ...

Disruptive changes made to docker configuration. Restarting docker service...
Created symlink /etc/systemd/system/sysvisor.service.wants/sysvisor-fs.service → /lib/systemd/system/sysvisor-fs.service.
Created symlink /etc/systemd/system/sysvisor.service.wants/sysvisor-mgr.service → /lib/systemd/system/sysvisor-mgr.service.
Created symlink /etc/systemd/system/multi-user.target.wants/sysvisor.service → /lib/systemd/system/sysvisor.service.
```

In case an error is observed above as a consequence of a missing
software dependency, proceed to download and install the missing
package(s) as indicated below. Once this requirement is satisfied,
Sysvisor's installation process will be automatically re-launched to
conclude this task.

Missing dependency output:

```
...
dpkg: dependency problems prevent configuration of sysvisor:
sysvisor depends on jq; however:
Package jq is not installed.

dpkg: error processing package sysvisor (--install):
dependency problems - leaving unconfigured
Errors were encountered while processing:
sysvisor
```

Install missing package by fixing (-f) system's dependency structures.

```
$ sudo apt-get install -f -y
```

Verify that Sysvisor's systemd units have been properly installed, and
associated daemons are properly running:

```
$ systemctl list-units -t service --all | grep sysvisor
sysvisor-fs.service                   loaded    active   running Sysvisor-fs component
sysvisor-mgr.service                  loaded    active   running Sysvisor-mgr component
sysvisor.service                      loaded    active   exited  Sysvisor General Service
```

## Sysvisor Logs

### sysvisor-mgr & sysvisor-fs

The Sysvisor daemons (i.e. sysvisor-fs and sysvisor-mgr) will log
information related to their activities in the
`/var/log/sysvisor-fs.log` and `/var/log/sysvisor-mgr.log` files
respectively. These logs should be useful during troubleshooting
exercises.

### sysvisor-runc

For sysvisor-runc, logging is handled as follows:

* When running Docker + sysvisor-runc, the sysvisor-runc logs are actually stored in
  a containerd directory such as:

  `/run/containerd/io.containerd.runtime.v1.linux/moby/<container-id>/log.json`

* When running sysvisor-runc directly, the sysvisor-runc will not log output by default.
  Use the `--log` option to sysvisor-runc to change this.

## Bind Mount Security Check Error

When creating a container with a bind mount, Docker may report the following error:

```bash
$ docker run --runtime=sysvisor-runc -it --mount type=bind,source=/some/path,target=/mnt/path debian:latest

docker: Error response from daemon: OCI runtime create failed: ... shiftfs mountpoint security check failed: path /some/path is not exclusively accessible to the root user or group ...
```

This means that the bind mount security check has failed. This occurs when running with Docker
userns remap disabled (the default configuration for Docker) and the source of the bind
mount does not meet Sysvisor's security requirements. Refer to [Docker Bind Mount Permissions](usage.md#docker-bind-mount-permissions)
for details on how to fix this.

## Bind Mount Permissions Error

When running a container with a bind mount, you may see that the files
and directories associated with the mount have `nobody:nouser`
ownership from within the container.

This typically occurs when the source of the bind mount is owned by
a user on the host that is different from the user on the host to which
the container's root user maps (recall that Sysvisor containers always
use the Linux user namespace and thus map the root user in the container
to a non-root user on the host).

If the container was created via Docker with userns-remap disabled
(the default configuration of Docker), then make sure that the bind
mount source has `root:root` ownership on the host.

If the container was created via Docker with userns-remap enabled,
then make sure that the bind mount source has the same owner
(user:group) as the Docker userns remap configuration.

Refer to [Docker Bind Mount Permissions](usage.md#docker-bind-mount-permissions) for further
details.

## Nestybox Shiftfs Module Error

When creating a container, the following error indicates that the Nestybox Shiftfs module
is required by Sysvisor but is not loaded in the Linux kernel:

```bash
# docker run --runtime=sysvisor-runc -it debian:latest
docker: Error response from daemon: OCI runtime create failed:  container requires uid shifting but error was found: nbox_shiftfs module is not loaded in the kernel
```

To solve this problem, load the Nestybox Shiftfs module as described [here](https://github.com/nestybox/nbox-shiftfs-external).

Note that normally the Sysvisor installer loads this module into the
kernel, so this error implies the module was somehow unloaded since
then.
