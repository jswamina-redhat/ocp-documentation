# Purpose

This document will provide a baseline understanding of what machine configs are and how they work in OpenShift Container Platform. 

# Introduction

In OpenShift, `MachineConfig` is a Kubernetes Custom Resource Definition (CRD) that manages the configuration of the operating system (OS) on OpenShift nodes.

To make changes to the operating systems running on OCP container platform nodes, you can create `MachineConfig` objects that are managed by the Machine Config Operator (MCO).

With MachineConfigs, you can manage and update:
- `systemd`
- `CRI-O`
- `kubelet`
- Kernel
- Network Manager
- Other system features

There are three daemon sets that OpenShift employs to simplify node management. These orchestrate operating system updates and configuration changes to the host by using standard Kubernetes-style constructs.

| Daemon Set | Function |
| ----------- | ----------- |
| `machine-config-controller` | Coordinates machine upgrades from the control plane, monitors all of the cluster nodes, and orchestrates their configuration updates |
| `machine-config-daemon` | Runs on each node in a cluster and updates a machine to configuration as defined by machine config and as instructed by the MachineConfigController. This daemon reads the machine configuration to see if it needs to do an OSTree update or if it must apply a series of systemd kubelet file changes, configuration changes, or other changes to the OS or OCP configuration. |
| `machine-config-serverr` | Provides the Ignition config files to control plane nodes as they join the cluster |

When a node does not fully match what the currently applied machine config specifies, it is in a state known as *configuration drift*. The Machine Config Daemon (MCD) regularly checks the nodes for configuration drift and when detected, will mark the node as `degraded` until an administrator corrects the node configuration.

[Note] A degraded node is online and operational, but cannot be updated

The MCD performs configuration drift detection upon each of the following conditions:
- When a node boots
- After any of the files (Ignition files and systemd drop-in units) specified in the machine config are modified outside of the machine config
- Before a new machine config is applied

[Info] If you apply a new machine config to the nodes, the MCD temporarily shuts down configuration drift detection. This shutdown is needed because the new machine config necessarily differs from the machine config on the nodes. After the new machine config is applied, the MCD restarts detecting configuration drift using the new machine config.

## MachineConfig Overview
MCO manages updates to `systemd`, `CRI-O`, and `Kubelet`, the kernel, Network Manager and other system features. With a `MachineConfig` CRD, you can write configuration files onto the host.

A `MachineConfig` can make a specific change to a file or service on the operating system of each system representing a pool of OpenShift Container Platform nodes.

The MachineConfig Operator applies changes to the operating systems in pools of machines. All OpenShift Container Platform clusters start with worker and control plane node pools. 

# Changes You Can Make with MachineConfigs

You can change the following components with the MCO:

| Syntax | Description |
| ----------- | ----------- |
| `config` | Create Ignition config objects to do things like modify files, systemd services, and other features on OCP - Configuration files: create or overwrite files in the `/var` or `/etc` directory - systemd units: Create and set the status of a `systemd` service or add to an existing `systemd` service by dropping in additional settings - users and groups: Change SSH keys in the `passwd` section post installation |
| `kernalArguments` | Add arguments to the kernel command line when OpenShift Container platform nodes boot |
| `kernalType` | Optionally identify a non-standard kernel to use instead of the standard |
| `fips` | Enable FIPS mode |
| `extenstions` | Extend RHCOS features by adding selected pre-packaged software. Available extensions include usbguard and kernel modules |
| Custom resources (for `ContainerRuntime` and `Kubelet`) | Outside for machine configs, MCO manages two special custom resources for modifying `CRI-O` container runtime settings (`ContainerRuntime` CR) and the Kubelet service (`Kubelet` CR) |

# Using MachineConfig Objects to Configure Nodes
Let’s go over a few examples of using a `MachineConfig` to configure nodes.

## Configuring `chrony` time service
You can set the time server used by the `chrony` time service (`chronyd`) by modifying the contents of the `chrony.conf` file and passing those contents to your nodes as a machine config.

**Procedure:**

1. Create a Butane config including the contents of the `chrony.conf` file:
EXAMPLE: Configure `chrony` workers on nodes, create a `99-worker-conry.bu` file
```
variant: openshift
version: 4.17.0
metadata:
  name: 99-worker-chrony 							        [1]
  labels:
    machineconfiguration.openshift.io/role: worker 			[2]
storage:
  files:
  - path: /etc/chrony.conf
    mode: 0644 								                [3]
    overwrite: true
    contents:
      inline: |
        pool 0.rhel.pool.ntp.org iburst 				    [4]
        driftfile /var/lib/chrony/drift
        makestep 1.0 3
        rtcsync
        logdir /var/log/chrony
```
EXPLANATION:

[1] [2] Substitute master for worker on control plane nodes

[3] Specify an octal value mode for the mode field in the machine config file

[4] Specify any valid, reachable time source, such as one provided by your DHCP server

2. Use Butane to generate a `MachineConfig` object file containing the configuration to be delivered to the nodes:
```
$ butane 99-worker-chrony.bu -o 99-worker-chrony.yaml
```

3. Apply the configuration via:
  a. If the cluster is not running yet (like in Baremetal IPI Installation), add the MachineConfig object file to the `<installation_directory>/openshift` directory, then create the cluster
  b. If the cluster is running, apply the file: `$ oc apply -f ./99-worker/chrony.yaml`

## Configuring `journald` settings
In this example, we will modify `journald` rate limiting settings in the /etc/systemd/journald.conf file and apply them to worker nodes. 

**Prerequisites:**
- Log into the cluster as a user with administrative privileges

**Procedure:**

1. Create a Butane config file, (ex: `40-worker-custom-journald.bu`), that includes an `/etc/systemd/journald.conf` file with the required settings:
```
variant: openshift
version: 4.17.0
metadata:
  name: 40-worker-custom-journald
  labels:
    machineconfiguration.openshift.io/role: worker
storage:
  files:
  - path: /etc/systemd/journald.conf
    mode: 0644
    overwrite: true
    contents:
      inline: |
        # Disable rate limiting
        RateLimitInterval=1s
        RateLimitBurst=10000
        Storage=volatile
        Compress=no
        MaxRetentionSec=30s
```

2. Use Butane to generate a `MachineConfig` object file, `40-worker-custom-journald.yaml`, containing the configuration to be delivered to the worker nodes:
```
$ butane 40-worker-custom-journald.bu -o 40-worker-custom-journald.yaml
```

3. Apply the machine configuration to the pool:
```
$ oc apply -f 40-worker-custom-journald.yaml
```

4. Verify that the new machine config is applied and that the nodes are not in a degraded state,(this may take a few minutes):
```
$ oc get machineconfigpool

## Example Output ##

NAME   CONFIG             UPDATED UPDATING DEGRADED MACHINECOUNT READYMACHINECOUNT UPDATEDMACHINECOUNT DEGRADEDMACHINECOUNT AGE
master rendered-master-35 True    False    False    3            3                 3                   0                    34m
worker rendered-worker-d8 False   True     False    3            1                 1                   0                    34m
```

5. To check that the change was applied, log in to a worker node:
```
$ oc get node | grep worker

## Example Output ##
ip-10-0-0-1.us-east-2.compute.internal   Ready    worker   39m   v0.0.0-master+$Format:%h$

$ oc debug node/ip-10-0-0-1.us-east-2.compute.internal

## Example Output ##
Starting pod/ip-10-0-141-142us-east-2computeinternal-debug ...
...
sh-4.2# chroot /host
sh-4.4# cat /etc/systemd/journald.conf
# Disable rate limiting
RateLimitInterval=1s
RateLimitBurst=10000
Storage=volatile
Compress=no
MaxRetentionSec=30s
sh-4.4# exit
```

# Summary

A Machine Config is a Kubernetes Custom Resource Definition that manages the configuration of the operating system on OpenShift nodes.

You change the following components with MCO:
- `config`
- `kernalArguments`
- `kernalType`
- `fips`
- `extensions`
- Custom resources (for `ContainerRuntime` and `Kubelet`)

There are three daemon sets orchestrate OS updates and configuration changes to the hosts:
- `machine-config-controller`
- `machine-config-daemon`
- `machine-config-server`

# Conclusion
This was a brief overview of what MachineConfigs are and how they work in OpenShift Container Platform. If you have any questions, please feel free to reach out to any of your Red Hat resources.

## Relevant Documentation
https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/machine_configuration/machine-config-index
https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/machine_configuration/machine-configs-configure
https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/machine_configuration/machine-configs-custom
