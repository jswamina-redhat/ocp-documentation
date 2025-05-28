# Purpose

This document should provide a baseline understanding of what network attachment definitions are and how they work in OpenShift Container Platform. 

# Introduction

In OpenShift Container Platform (OCP), Network Attachment Definitions (NADs) are custom resources that allow pods and virtual machines to access specific, non-default networks.

**Bridge** → Allows pods to connect to bridge network on the host. This allows pods on the same host to communicate with each other and the host itself.

Here’s how the bridge CNI plugin works:
1. Bridge Creation: The `bridge` plugin creates a virtual network bridge on the host network namespace
2. Virtual Ethernet Pair: For each pod, a pair of virtual Ethernet interfaces is created where one end is placed on the pod’s network namespace and the other is connected to the bridge
3. IP Address Assignment: IP address is assigned to the the virtual Ethernet interface on the pod’s network namespace
4. Host Connectivity: The bridge can be configured as a gateway allowing pods to communicate with the host

**MacVLAN** → Facilitates binding a physical interface directly to multiple namespaces without the need for a bridge

**SR-IOV** → Single Root I/O Virtualization allows a single PCIe device to appear as multiple virtual devices

**VLANs** → Virtual Local Area Network is a virtualized connection that connects multiple devices and network nodes from different LANs into one logical network

This custom resource exposes layer-2* devices to a specific namespace in your OpenShift Virtualization layer.

Network admins can create NADs to provide existing layer-2 networking to pods and virtual machines.

*A layer-2 device is a device that makes a forwarding decision on a physical address. For example, a bridge or a switch.

# Configuring Network Attachment Definitions

NAD’s allow multiple network interfaces for VMs, enabling hybrid connectivity, virtual local area networks (VLANs), and high-performance network scenarios.

This Kubernetes CustomResourceDefinition (CRD) is used to define additional network interfaces for pods or virtual machines.

## OpenShift Virtualization NAD Architecture

The NAD architecture for the following example can be broken down into the following:
- **Primary network**: OVN-Kubernetes
- **Multus CNI**: Manages secondary networks and interfaces
- **Plug-ins**: Specific configurations for MacVLAN, SR-IOV, or other setups

# Attaching a Virtual Machine to a Linux Bridge Network

OpenShift Virtualization is installed with a single, internal pod network by default.

You must create a Linux bridge network attachment definition in order to connect additional networks.

## Creating a Linux Bridge Network Attachment Definition

You can create a linux bridge NAD using the web console or CLI.

**GUI Procedure**:
1. In the web console, click `Networking` > `Network Attachment Definitions`
2. Click `Create Network Attachment Definition`
  a. Must be in the same namespace as the pod or virtual machine
3. Enter a unique `Name`
4. Click the `Network Type` list and select `CNV Linux bridge`
5. Enter a name for the bridge in the `Bridge Name` field
6. Optional: Enter `VLAN Tag Number`
7. Optional: Select `MAC Spoof Check` to provide security against spoofing
8. Click `Create`

**CLI Procedure**:
1. Create a NAD in the same namespace as the virtual machine
2. Add the VM to the network attachment definition

`YAML`
```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: <bridge-network> 						                                     # [1]
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/<bridge-interface> 
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "<bridge-network>", 					                                  # [2]
    "type": "cnv-bridge", 						                                      # [3]
    "bridge": "<bridge-interface>", 				                                  # [4]
    "macspoofchk": true,
    "vlan": 1 
  }'
```
**Explanation**:

[1] The name for the `NetworkAttachmentDefinition` object

[2] The name for the configuration. It is recommended to match the configuration name to the `name` value of the network attachment definition

[3] The actual name for the container Network Interface (CNI) plugin that provides the network for this NAD

[4] The name of the Linux bridge configured node

**Create the network attachment definition**:
```
$ oc create -f <network-attachment-definition>.yaml
```

**Verify that the NAD was created**:
```
$ oc get network-attachment-definition <bridge-network>
```

## Configuring the Virtual Machine for a Linux Bridge Network

> **_INFO:_**  Prerequisite: Shut down the VM before editing the configuration. If you edit a running VM, you must restart the VM for changes to take effect

**CLI YAML Procedure**:
1. Create/edit a configuration of a virtual machine that you want to connect to the bridge network
2. Add the bridge interface and the network attachment definition to the VM’s configuration

**Example Configuration**:
`YAML`
```
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
    name: <example-vm>
spec:
  template:
    spec:
      domain:
        devices:
          interfaces:
            - masquerade: {}
              name: <default>
            - bridge: {}
              name: <bridge-net> 						            # [1]
...
      networks:
        - name: <default>
          pod: {}
        - name: <bridge-net> 						                # [2]
          multus:
            networkName: <network-namespace>/<a-bridge-network> 	# [3]
...
```
**Explanation**:

[1] The name of the bridge interface

[2] The name of the network. This value must match the `name` value of the corresponding `spec.template.spec.domain.devices.interfaces` entry

[3] The name of the network attachment definition under `spec.template.spec.networks`, prefixed by the namespace where it exists. The namespace must be either the `default` namespace of the same namespace where the VM is to be created

Apply the configuration:
`$ oc apply -f <example-vm>.yaml`

> **_INFO:_**  If the VM was running as you edited its configuration, make sure you restart the VM to apply the new configuration.

## Creating a Linux Node Network Configuration

Use a `NodeNetworkConfigurationPolicy` manifest `YAML` file to create the Linux bridge

`YAML`
```
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br1-eth1-policy 						         # [1]
spec:
  desiredState:
    interfaces:
      - name: br1 							             # [2]
        description: Linux bridge with eth1 as a port 
        type: linux-bridge 					             # [3]
        state: up 							             # [4]
        ipv4:
          enabled: false 						         # [5]
        bridge:
          options:
            stp:
              enabled: false 					         # [6]
          port:
            - name: eth1						         # [7]
```
Run `$ oc apply -f <file_name>.yaml` to apply the configuration.

Explanation:

[1] Name of policy

[2] Name of the interface

[3] The type of interface. This example creates a bridge

[4] The requested state for the interface after creation

[5] Disables IPv4 in this example

[6] Disables STP in this example

[7] The node NIC to which the bridge is attached

## Creating and Using Network Attachment Definitions

We will go over three (3) example configurations:
- MacVLAN NAD configuration
- SR-IOV NAD configuration
- VLAN configuration

### MacVLAN NAD Configuration
`YAML`
```
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
 name: macvlan-network
 namespace: my-namespace
spec:
 config: '{
   "cniVersion": "0.3.1",
   "type": "macvlan",
   "master": "eth0",
   "mode": "bridge",
   "ipam": {
     "type": "static",
     "addresses": [
       {
         "address": "192.168.1.100/24",
         "gateway": "192.168.1.1"
       }
     ]
   }
 }'
```
Run `$ oc apply -f <file_name>.yaml` to apply the configuration.

Attach the MacVLAN NAD to a VM by adding the NAD to your VM specification under `networks` and `interfaces`.

`YAML`
```
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
 name: my-vm
 namespace: my-namespace
spec:
 template:
   spec:
     domain:
       devices:
         interfaces:
         - name: default
           masquerade: {}
         - name: macvlan-net
           bridge: {}
     networks:
     - name: default
       pod: {}
     - name: macvlan-net
       multus:
         networkName: my-namespace/macvlan-network
```
Run `$ oc apply -f <file_name>.yaml` to apply the configuration.

This example sets up a MacVLAN-based secondary network for a VM. This is often used to segment traffic within the same physical network.

### SR-IOV NAD Configuration
`YAML`
```
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
 name: sriov-network
 namespace: my-namespace
spec:
 config: '{
   "cniVersion": "0.3.1",
   "type": "sriov",
   "ipam": {
     "type": "host-local",
     "subnet": "192.168.2.0/24",
     "rangeStart": "192.168.2.100",
     "rangeEnd": "192.168.2.200",
     "gateway": "192.168.2.1"
   }
 }'
```
Run `$ oc apply -f <file_name>.yaml` to apply the configuration.

Attach SR-IOV NAD to a VM and update the VM spec to include the SR-IOV interface:

`YAML`
```
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
 name: sriov-vm
 namespace: my-namespace
spec:
 template:
   spec:
     domain:
       devices:
         interfaces:
         - name: default
           masquerade: {}
         - name: sriov-net
           bridge: {}
     networks:
     - name: default
       pod: {}
     - name: sriov-net
       multus:
         networkName: my-namespace/sriov-network
```
Run `$ oc apply -f <file_name>.yaml` to apply the configuration.

SR-IOV provides near-native network performance by dedicating a virtual function (VF) from a physical network interface controller (NIC) to a VM.

### VLAN Configuration
`YAML`
```
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vlan-network
  namespace: my-namespace
spec:
  config:
    '{
      "cniVersion": "0.3.1",
      "type": "vlan",
      "vlanId": 100,
      "master": "eth0",
      "ipam":
        {
          "type": "static",
          "addresses":
            [{ "address": "10.10.1.100/24", "gateway": "10.10.1.1" }],
        },
    }'
```
Run `$ oc apply -f <file_name>.yaml` to apply the configuration.

Attach VLAN `NetworkAttachmentDefinition` to a Virtual Machine:

`YAML`
```
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vlan-vm
  namespace: my-namespace
spec:
  template:
    spec:
      domain:
        devices:
          interfaces:
            - name: default
              masquerade: {}
            - name: vlan-net
              bridge: {}
        networks:
          - name: default
            pod: {}
          - name: vlan-net
            multus:
              networkName: my-namespace/vlan-network
```
Run `$ oc apply -f <file_name>.yaml` to apply the configuration.

VLANs allow logical segmentation of networks, which can be configured using Multus and a VLAN container network interface (CNI) plug-in.

# Summary
- Network Attachment Definitions (NADs) are custom resources that allow pods and virtual machines to access specific, non-default networks. Examples include:
  - Bridge
  - MacVLAN
  - SR-IOV
  - VLANs
- You can attach Virtual Machines to Linux Bridge Networks
- You can create a NAD using: `oc create -f <network-attachment-definition>.yaml`
- You can verify the creation of a NAD using: `oc get network-attachment-definition <bridge-network>`

# Conclusion
This was a brief overview of the Network Attachment Definition (NAD) custom Kubernetes resource and how it pertains to OpenShift Virtualization. This should function as a primer in understanding basic OCPVirt networking.

## Relevant Documentation
https://developers.redhat.com/articles/2024/12/19/how-configure-network-attachment-definitions#how_to_create_and_use_nads

https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/virtualization/virtual-machines#virt-creating-vms-from-rh-images-overview

https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/networking/index

https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/networking/multiple-networks#understanding-multiple-networks
