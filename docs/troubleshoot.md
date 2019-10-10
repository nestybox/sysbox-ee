Sysbox Troubleshooting
========================

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
```

Or if your Docker daemon is configured with [userns-remap](usage.md#interaction-with-docker-userns-remap), the
expected output is:

```console
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

Both mean that the installation completed successfully.

In case an error is observed above as a consequence of a missing
software dependency, proceed to download and install the missing
package(s) as indicated below. Once this requirement is satisfied,
Sysbox's installation process will be automatically re-launched to
conclude this task.

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

When this file is changed, the Docker daemon needs to be restarted:

```console
# systemctl restart docker.service
```

**Note:** The sysbox installer automatically configures the `/etc/docker/daemon.json`
file to add the `sysbox-runc` runtime to it, and restarts the Docker daemon.

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

## Ubuntu Shiftfs Module Not Present

When creating a system container, the following error indicates that
the Ubuntu shiftfs module is required by Sysbox but is not loaded
in the Linux kernel:

```console
# docker run --runtime=sysbox-runc -it debian:latest
docker: Error response from daemon: OCI runtime create failed:  container requires uid shifting but error was found: shiftfs module is not loaded in the kernel
```

The Ubuntu shiftfs module is required when Sysbox detects that Docker is
running without userns-remap (Docker's default configuration).

The error likely means you are running Sysbox on an older Ubuntu kernel. See [here](../README.md#supported-linux-distros)
for the list of Linux distros supported by Sysbox.

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

## Failed to Setup Docker Volume Manager Error

When creating a system container, Docker may report the following error:

```console
docker run --runtime=sysbox-runc -it ubuntu:latest
docker: Error response from daemon: OCI runtime create failed: failed to setup docker volume manager: host dir for docker store /var/lib/sysbox/docker can't be on ..."
```

This means that directory `/var/lib/sysbox` is on a filesystem not supported by Sysbox.

This directory must be on one of the following filesystems:

   * ext4
   * btrfs

The same requirement applies to the `/var/lib/docker` directory.

This is normally the case for vanilla Ubuntu installations.
