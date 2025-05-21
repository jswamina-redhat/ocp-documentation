# Purpose

This will serve as a brief introduction to OpenShift Virtualization, the MTV Operator, and an installation guide.

# Introduction

MTV, or Migration Toolkit for Virtualization Operator, is an operator that facilitates the movement of VM workloads over to Openshift Virtualization. 

You can migrate virtual machines from VMware vSphere or Red Hat Virtualization to OpenShift Virtualization.

Here are the types of virtual machines are supported:
- VMware vSphere
- OVAs created by VMware vSphere
- Red Hat Virtualization (RHV)
- OpenStack

Migrations can be done via a **cold migration** or **warm migration**.

**Cold Migration**
This is the default migration type. The source VMs are shut down while the data is copied.

**Warm Migration**
With this migration strategy, most of the data is copied during the *precopy* stage while the source VMs are running.

Then the VMs are shut down and the remaining data is copied during the *cutover* stage.

**Precopy Stage**
- VMs are not shut down
- VM disks are copied incrementally using changed block tracking (CBT) snapshots
- Runs until the cutover stage is started manually or is scheduled to start

**Cutover Stage**
- VMs are shut down during this stage
- Remaining data is migrated (data stored in RAM is not migrated)
- Can start manually using the MTV console or schedule a cutover time in the `Migration` manifest

# Installing the MTV Operator

## OCP Web Console

One option to install the MTV Operator is to use the OpenShift Container Platform web console

You will need to have the OpenShift Virtualization Operator installed and be logged in as a user with `cluster-admin` permissions.

**Procedure**
1. Navigate to `Operators` > `Operator Hub` in the OCP web console
2. Search for `mtv-operator`
3. Click `Migration Toolkit for Virtualization Operator` > `Install`
4. Click `Install` on the `Install Operator` page
5. Navigate to `Operators` > `Installed Operators` to verify that the MTV operator shows up in the `openshift-mtv` project with the `Succeeded` status
6. Click on the MTV operator
7. Locate the `ForkliftController` under `Provided APIs` and click `Create Instance`
8. Click `Create`
9. Navigate to `Workloads` > `Pods` to verify that the MTV pods are running

**To obtain the MTV web console URL, do the following:**
1. Navigate to `Networking` > `Routes`
2. Select the `openshift-mtv` project
3. Click the URL for the `forklift-ui service`, this will open the login page for the MTV console

## Command Line INterface

You can also install the MTV operator using the CLI. Make sure you are logged in as a user with `cluster-admin` permissions.

**Procedure**
1. Create the `openshift-mtv` project:
```
apiVersion: project.openshift.io/v1
kind: Project
metadata:
  name: openshift-mtv
```
`oc apply -f <file_name>.yaml`

2. Create an `OperatorGroup` custom resource (CR) called `migration`:
```
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: migration
  namespace: openshift-mtv
spec:
  targetNamespaces:
    - openshift-mtv
```
`oc apply -f <file_name>.yaml`

3. Create the `Subscription` CR for the Operator:
```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: mtv-operator
  namespace: openshift-mtv
spec:
  channel: release-v2.2.0
  installPlanApproval: Automatic
  name: mtv-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: "mtv-operator.2.2.0"
```
`oc apply -f <file_name>.yaml`

4. Create the `ForkliftController` CR:
```
apiVersion: forklift.konveyor.io/v1beta1
kind: ForkliftController
metadata:
  name: forklift-controller
  namespace: openshift-mtv
spec:
  olm_managed: true
```
`oc apply -f <file_name>.yaml`

5. Verify the MTV pods are running:

```$ oc get pods -n openshift-tv```

**To build a source VMware Host you will need the following:**
- VCenter host URL, service account and password
- vddk image path (i.e. quay.io/gm_si_rhosv/vddk:latest)
- VCenter certificates

**To obtain the MTV web console URL from the CLI, do the following:**
1. Obtain the MTV web console URL:
```
oc get route virt -n openshift-mtv \
  -o custom-columns=:.spec.host
```
Example Output: `https://virt-openshift-mtv.apps.cluster.openshift.com`

2. Launch a browser and navigate to the MTV web console with the obtained URL

# Summary
- The Migration Toolkit for Virtualization Operator (MTV) facilitates the movement of VM workloads over to Openshift Virtualization
- You can migrate via cold or warm migrations
- You can install the operator via the OCP web console or the CLI

# Conclusion
You should now know how to install the MTV operator and what options you have when it comes to migrating workloads over to OpenShift Container Platform.

## Relevant Documentation

https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.5/html/installing_and_using_the_migration_toolkit_for_virtualization/about-mtv

https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.5/html/installing_and_using_the_migration_toolkit_for_virtualization/installing-the-operator
