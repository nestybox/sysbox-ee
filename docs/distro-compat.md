# Sysbox Distro Compatibility

## Contents

-   [Supported Linux Distros](#supported-linux-distros)
-   [Ubuntu Support](#ubuntu-support)
    -   [Sysbox on Older Ubuntu Kernels](#sysbox-on-older-ubuntu-kernels)
    -   [Distro Compatibility Summary](#distro-compatibility-summary)
    -   [Upgrading the Ubuntu Kernel](#upgrading-the-ubuntu-kernel)
        -   [Bionic Beaver](#bionic-beaver)
        -   [Disco Dingo](#disco-dingo)

## Supported Linux Distros

Sysbox relies on functionality that is currently only present in
Ubuntu kernels. Thus, Ubuntu is the only distro supported at this
time.

## Ubuntu Support

Sysbox requires very recent Ubuntu kernels:

-   Ubuntu 19.10 "Eoan"
-   Ubuntu 19.04 "Disco" (kernel upgrade >= 5.0.0-21.22)
-   Ubuntu 18.04 "Bionic" (kernel upgrade >= 5.0)

If your kernel is not one of these, you can still use Sysbox
using the alternative approach described in the next section.

**NOTES:**

1) Canonical generates flavors of these kernels for desktop, server,
and cloud. The [cloud images](https://cloud-images.ubuntu.com/) do
not yet include the kernel functionality required by Sysbox
(unfortunately). Thus, if you want to use Sysbox on a cloud VM, you
must install the Ubuntu desktop or server flavors on the VM, or use
the alternative approach described in the next section.

2) If you have a Ubuntu desktop or server image but need to upgrade
the kernel in order to meet the requirements listed above, see
[here](#upgrading-the-ubuntu-kernel) for suggestions on how to do this.

3) If you launch a system container with Docker + Sysbox, and you get
an error such as

```console
# docker run --runtime=sysbox-runc -it debian:latest
docker: Error response from daemon: OCI runtime create failed: container requires user-ID shifting but error was found: shiftfs module is not loaded in the kernel. Update your kernel to include shiftfs module or enable Docker with userns-remap. Refer to the Sysbox troubleshooting guide for more info: unknown
```

it means that your kernel does not meet the requirements listed above
(specifically your distro does not have the `shiftfs` module that is
present in recent Ubuntu desktop and server kernels). In this case you
must either upgrade your kernel or use the alternative approach
described in the next section.

### Sysbox on Older Ubuntu Kernels

If your distro is Ubuntu but the kernel version is not as recent as
those listed in the prior section, it's still possible to use
Sysbox. However, doing so requires two things:

1) You must have Ubuntu Bionic or later (e.g., Ubuntu Bionic, Disco,
   Cosmic, or Eoan). Any kernel version of these releases will do.

2) You must change the configuration of the Docker daemon:

-   After installing Sysbox, add the `userns-remap` line to the
    `/etc/docker/daemon.json` file as shown below:

    ```console
    # cat /etc/docker/daemon.json
    {
      "userns-remap": "sysbox",
      "runtimes": {
        "sysbox-runc": {
          "path": "/usr/local/sbin/sysbox-runc"
        }
      }
    }
    ```

    This tells Docker to enable the user namespace in all containers
    and map the root user in the container to the subid range of
    user `sysbox`, as defined in the `/etc/subuid` and `/etc/subgid` files.

-   Restart the Docker daemon:

    ```console
    # sudo systemctl restart docker.service
    ```

-   Note that both of the actions above require `root` privileges on
    the host.

When Docker is configured this way, Sysbox works in what we call
"Docker userns-remap mode". It's a mode that offers strong container
isolation, but not as strong as when using the latest Ubuntu kernels
with the shiftfs module as described in the prior section (where
Sysbox works in "Exclusive userns-remap mode". See
[here](usage.md#system-container-isolation-modes) for more info on
Sysbox isolation modes.

### Distro Compatibility Summary

Here is a summary of the distro requirements:

| Distro Name         | Kernel           | Requires Docker in Userns-Remap Mode |
| ------------------- | ---------------- | ------------------------------------ |
| Ubuntu 19.10 Eoan   | Any              | No                                   |
| Ubuntu 19.04 Disco  | >= 5.0.0-21.22   | No                                   |
|                     | &lt; 5.0.0-21.22 | Yes                                  |
| Ubuntu 18.10 Cosmic | Any              | Yes                                  |
| Ubuntu 18.04 Bionic | >= 5.0           | No                                   |
|                     | &lt; 5.0         | Yes                                  |

### Upgrading the Ubuntu Kernel

In case you need to upgrade your machine's Linux kernel to meet Sysbox
kernel version requirements, here are some suggestions. Refer to the
Ubuntu documentation online for further info.

#### Bionic Beaver

For Bionic Beaver, we recommend using Ubuntu's [LTS-enablement](https://wiki.ubuntu.com/Kernel/LTSEnablementStack)
package:

```console
$ sudo apt-get update && sudo apt install --install-recommends linux-generic-hwe-18.04 -y
```

#### Disco Dingo

For Disco Dingo, we recommend simply upgrading the distro:

```console
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt-get dist-upgrade
$ reboot
```
