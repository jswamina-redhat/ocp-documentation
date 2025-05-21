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
| `kernalArguments` | Text |
| `kernalType` | Title |
| `fips` | Title |
| `extenstions` | Title |
| Custom resources (for `ContainerRuntime` and `Kubelet`) | Title |
