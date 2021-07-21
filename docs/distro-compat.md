# Sysbox-EE Distro Compatibility

## Contents

-   [Supported Linux Distros](#supported-linux-distros)
-   [Kernel Upgrade Procedures](#kernel-upgrade-procedures)

## Supported Linux Distros

The following table summarizes the supported Linux distros, the installation
methods supported, and any other requirements:

| Distro / Release      | Package Install | K8s Install | Kernel Upgrade | Other |
| --------------------- | :-------------: | :---------: | :----: | :----: |
| Ubuntu Bionic (18.04) | ✓ | ✓ | [Maybe](#ubuntu-kernel-upgrade) | |
| Ubuntu Focal  (20.04) | ✓ | ✓ | No                              | |
| Debian Buster (10)    | ✓ |   | [Yes](#debian-kernel-upgrade)   | |
| Debian Bullseye (11)  | ✓ |   | No                              | |
| Flatcar               | N/A |   | No                            | More details [here](https://github.com/nestybox/sysbox-flatcar-preview) |

**NOTES:**

-   "Package install" means a Sysbox-EE package is available for that distro. See [here](../README.md#installation) for more.

-   "K8s-install" means you can deploy Sysbox-EE on a Kubernetes worker node based on that distro. See [here](../README.md#installation) for more.

-   "Kernel upgrade" means a kernel upgrade may be required (Sysbox-EE requires a fairly new kernel). See [below](#kernel-upgrade-procedures) for more.

## Kernel Upgrade Procedures

### Ubuntu Kernel Upgrade

If you have a relatively old Ubuntu 18.04 release (e.g. 18.04.3), you need to upgrade the kernel to >= 5.0.

We recommend using Ubuntu's [LTS-enablement](https://wiki.ubuntu.com/Kernel/LTSEnablementStack) package to do the upgrade as follows:

```console
$ sudo apt-get update && sudo apt install --install-recommends linux-generic-hwe-18.04 -y

$ sudo shutdown -r now
```

### Debian Kernel Upgrade

This one is only required when running Debian Buster.

```console
$ # Allow debian-backports utilization ...

$ echo deb http://deb.debian.org/debian buster-backports main contrib non-free | sudo tee /etc/apt/sources.list.d/buster-backports.list

$ sudo apt update

$ sudo apt install -t buster-backports linux-image-amd64

$ sudo shutdown -r now
```

Refer to this [link](https://wiki.debian.org/HowToUpgradeKernel) for more details.
