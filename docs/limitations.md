# Sysbox Limitations

This document describes current Sysbox functional limitations.

## Contents

## Creating User Namespaces inside a System Container

Sysbox does not currently support unsharing a user-namespace inside a
system container and mounting procfs.

That is, executing the following instruction inside a system container
is not supported:

```
unshare -U -i -m -n -p -u -f --mount-proc -r bash
```

The reason this is not yet supported is that Sysbox is not currently
capable of ensuring that the procfs mounted inside the unshared
namespace is within the context of the procfs mounted in the system
container.

## Running inner Docker with userns-remap enabled

For the same reason described in the prior section, Sysbox does not
currently support configuring a Docker engine inside a system
container with "userns-remap".
