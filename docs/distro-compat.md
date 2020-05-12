# Sysbox Distro Compatibility

## Contents

-   [Supported Linux Distros](#supported-linux-distros)
-   [Ubuntu Support](#ubuntu-support)

## Supported Linux Distros

Sysbox relies on functionality that is currently only present in Ubuntu Linux
and as a result Ubuntu is the only distro supported at this time. We are working
on adding support for more distros.

## Ubuntu Support

Sysbox requires very recent Ubuntu versions:

-   Ubuntu 20.04 "Focal Fossa"
-   Ubuntu 19.10 "Eoan Ermine"
-   Ubuntu 18.04 "Bionic Beaver" (with kernel upgrade >= 5.0)

These versions carry some new Linux kernel features, including a module called
"shiftfs", which Sysbox uses to create the system containers.

If your Ubuntu version does not carry the shiftfs module, you'll see this error
when launching a container:

```console
# docker run --runtime=sysbox-runc -it debian:latest
docker: Error response from daemon: OCI runtime create failed: container requires user-ID shifting but error was found: shiftfs module is not loaded in the kernel. Update your kernel to include shiftfs module or enable Docker with userns-remap. Refer to the Sysbox troubleshooting guide for more info: unknown
```

Don't worry: you can still use Sysbox, but it requires that you configure Docker
with "userns-remap" (see the next section).

**NOTES:**

1) Canonical generates Ubuntu editions for desktop, server, and cloud VMs. The
[cloud editions](https://cloud-images.ubuntu.com/) may not carry the the shiftfs
module. If you want to use Sysbox on a cloud VM and you hit the error shown
above, you must either configure Docker with userns-remap as shown in the next
section, or install the Ubuntu desktop or server edition in the VM.

2) If you have a Ubuntu 18.04 (Bionic Beaver) and want to upgrade the kernel to >= 5.0,
we recommend using Ubuntu's [LTS-enablement](https://wiki.ubuntu.com/Kernel/LTSEnablementStack)
package to do the upgrade:

```console
$ sudo apt-get update && sudo apt install --install-recommends linux-generic-hwe-18.04 -y
```

### Using Sysbox on kernels without the shiftfs module

If your Ubuntu version does not carry the shiftfs module, using Sysbox requires
that you configure Docker in "userns-remap" mode.

This is done by modifying the configuration of the Docker daemon and restarting it:

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

    This tells Docker to use the Linux user namespace in all containers
    and map the root user in the container to the subid range of
    user `sysbox`, as defined in the `/etc/subuid` and `/etc/subgid` files.

-   Then restart the Docker daemon (make sure any running containers are stopped):

    ```console
    # sudo docker stop $(docker ps -aq)
    # sudo systemctl restart docker.service
    ```

Note that the actions above require `root` privileges on the host.

When Docker is configured this way, Sysbox no longer needs the shiftfs
module. But there are a couple of drawbacks:

-   Container isolation, while strong, is reduced compared to using shiftfs (see
    [here](usage.md#system-container-isolation-modes) for more info on this).

-   Configuring Docker this way places a few functional limitations on regular Docker containers
    (those launched with Docker's default runc), as described in this [Docker document](https://docs.docker.com/engine/security/userns-remap).
