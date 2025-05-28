# Purpose
This document outlines the required steps for adding an OpenShift cluster to Red Hat Advanced Cluster Management (RHACM).

# Steps for Adding a Cluster to RHACM

Log into your RHACM cluster

Navigate to `Infrastructure` > `Clusters` > `Import cluster`

### `Details` Page

1. Name: Input the name of your cluster
2. Cluster set: `<>`
3. Additional labels: 
  - `env=prod` for production environments
  - `env=lab` for lab environments
  - `env=preprod` for preprod environments
4. Import mode: Run import commands manually

### `Automation` page
1. Automation template: `none` > click `Next`

### `Review` page
1. Review and validate info
2. Click `Generate` command
3. Copy the generated command

# Running the Copied Command

You will need to run the copied command on the target cluster (the cluster you are trying to add to RHACM).

You can confirm that you are on the correct cluster by running `oc cluster-info`. This will return the api of the cluster you are on.

Additionally, make sure to run `oc whoami` to confirm you are logged in as your personal user on the appropriate cluster. 

If not, run `oc login -u <USER> <OCP_API>` then input your password when prompted.

While logged in as your user on the target cluster, paste the copied command from above and run.

You will see multiple Kubernetes resources being created, it will take a little bit of time for the cluster to be added to RHACM.

# Verification

### On the Target Cluster
1. Go to `Workloads` > `Pods`
2. Change the namespace to `open-cluster-management-agent`
3. Verify that there are pods that running

### On the RHACM Cluster
1. Navigate to `Infrastructure` > `Clusters` > `Cluster list`
2. Verify that you see the newly added cluster
3. Check for a `Ready` status for the cluster
4. Click on the cluster and verify everything under the `Nodes` and `Add-ons` tab has the `Available` status

### References
[1](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.2/html/manage_cluster/importing-a-target-managed-cluster-to-the-hub-cluster#importing-a-cluster)
