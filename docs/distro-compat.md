# Sysbox Distro Compatibility

## Contents

-   [Supported Linux Distros](#supported-linux-distros)
-   [Ubuntu Support](#ubuntu-support)

## Supported Linux Distros

Sysbox relies on functionality that is currently only present in Ubuntu Linux.
As a result Ubuntu is the only distro supported at this time. We are working
on adding support for more distros.

## Ubuntu Support

Sysbox requires recent Ubuntu versions:

-   Ubuntu 20.04 "Focal Fossa"
-   Ubuntu 19.10 "Eoan Ermine"
-   Ubuntu 18.04 "Bionic Beaver" (with kernel upgrade >= 5.0)

These versions carry some new Linux kernel features that Sysbox relies on to
create the system containers.

**NOTE:** If you have Ubuntu 18.04 (Bionic Beaver), you need to upgrade the kernel to >= 5.0.
We recommend using Ubuntu's [LTS-enablement](https://wiki.ubuntu.com/Kernel/LTSEnablementStack)
package to do the upgrade as follows:

```console
$ sudo apt-get update && sudo apt install --install-recommends linux-generic-hwe-18.04 -y
```
