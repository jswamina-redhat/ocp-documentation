# Purpose

This document will provide you with the necessary background information and installation instructions on the OCP Virt Operator.

# Introduction

Red Hat OpenShift Virtualization is an add-on to Red Hat OpenShift Container Platform.

Red Hat OpenShift Virtualization is an add-on to Red Hat OpenShift Container Platform.

OCP Virtualization adds new objects to your OpenShift Container Platform cluster via Kubernetes custom resources to enable virtualization tasks.

## Key Virtualization Tasks

**Creating and managing Linux and Windows virtual machines:**

OCP Virt provides a UI wizard to easily create Linux and Windows virtual machines.

You can also create VMs by providing YAML manifests or from virtual machine templates in the web console or on the command line.

**Importing and cloning existing virtual machines:**

Have already existing virtual machines that you need to keep as you transition? OpenShift Virtualization allows you to import existing VMs from VMware vSphere or Red Hat Virtualization.

As well as enabling importing of existing virtual disk images from HTTP, HTTPS, and registry endpoints.

**Live migrating virtual machines between nodes:**

With live migration, you can move a running virtual machine instance to another node in the cluster without interrupting the virtual workload or access

# Installing the Openshift Virtualization Operator

You can deploy the OpenShift Virtualization Operator by using the OpenShift Container Platform web console or command line.

## OCP Web Console

[!] Make sure you are logged in as a user with `cluster-admin` permissions

**Procedure**
1. Navigate to `Operators` > `OperatorHub`
2. Search for **“Virtualization”**
3. Select the OpenShift Virtualization Operator tile with the Red Hat source label
4. Click `Install`
5. On the Install Operator page:→ Select `stable` from the list of available Update Channel options to ensure you download a compatible operator version with your OCP version

 → For `Installed Namespace` ensure that the `Operator recommended namespace` option is selected to ensure it installs in the mandatory `openshift-cnv namespace` (this is automatically created if it does not exist)

 [!] If the OCP Virtualization operator is installed in a namespace other than openshift-cnv the installation will fail

 6. Click `Install` to make the Operator available to the `openshift-cnv` namespace
 7. When the Operator installs successfully, click `Create HyperConverged`
 8. Click `Create` to launch OpenShift Virtualization

**Verification**
To verify successful installation, navigate to the `Workloads` > `Pods` page and monitor the OpenShift Virtualization pods until they are all Running.

Once all the pods display the Running state, you can use OpenShift Virtualization! 🎉

## Command Line Interface

[!] Make sure you are logged in as a user with cluster-admin permissions

**Procedure**
1. First configure the necessary `Namespace`, `OperatorGrou`p, and `Subscription` objects by applying the single manifest to your cluster
```
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-cnv
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: kubevirt-hyperconverged-group
  namespace: openshift-cnv
spec:
  targetNamespaces:
    - openshift-cnv
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: hco-operatorhub
  namespace: openshift-cnv
spec:
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  name: kubevirt-hyperconverged
  startingCSV: kubevirt-hyperconverged-operator.v4.17.7
  channel: "stable"
```
Once created, save the file and run the following command: `oc apply -f <file_name>.yaml`

2. Create the `HyperConverged` (step 7 in the Web Console method) by creating the following manifest and applying it to your cluster to deploy the OpenShift Virtualization Operator:
```
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
  namespace: openshift-cnv
spec:
```
Save the file and run the following command: `oc apply -f <file_name>.yaml`

**Verification:**
Ensure that OpenShift Virtualization deployed successfully by watching the PHASE of the cluster service version (CSV) in the `openshift-cnv` namespace:

`watch oc get csv -n openshift-cnv`

Example output of a successful deployment:

```
NAME                                      DISPLAY                    VERSION   REPLACES   PHASE
kubevirt-hyperconverged-operator.v4.17.7   OpenShift Virtualization   4.17.7                Succeeded
```

# Summary

- The Red Hat OpenShift Virtualization Operator allows you to run and manage virtual machine workloads alongside container workloads
- You can install the operator via the OCP web console or the CLI

# Conclusion

You should now be able to install the Red Hat OpenShift Virtualization operator and begin running virtual machine workloads on OpenShift Container Platform

## Relevant Documentation

https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/virtualization/installing#installing-virt

https://catalog.redhat.com/software/container-stacks/detail/5ec53f218b6f188e53644c4f
