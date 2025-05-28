# Purpose
This document outlines how to access an OpenShift node for various tasks such as debugging.

# Accessing OCP Nodes

The best practice method for accessing OpenShift node is using oc debug, for example:

```
oc debug node/<node_name>
```

Alternatively, one can SSH into the OpenShift node from the bastion host. The SSH private keys are stored on the bastion host at `~$HOME/.ssh/id_ed25519`.

If the SSH private key is lost, the authorized SSH keys can be updated on the cluster after the installation. 

Refer to [Solution 3868301: How to update ssh keys after installation in Openshift 4?](https://access.redhat.com/solutions/3868301)

As a last resort, it is possible to break into the OpenShift node by booting it into a single user mode. This requires adding single to the kernel command-line.

See [Solution 5500131: How to start Red Hat CoreOS (RHCOS) in emergency mode in OpenShift Container Platform 4](https://access.redhat.com/solutions/5500131)
