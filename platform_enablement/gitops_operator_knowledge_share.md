Red Hat OpenShift GitOps uses Argo CD to manage specific cluster-scoped resources, including cluster Operators, optional Operator Lifecycle Manager (OLM) Operators, and user management.

# Installing Red Hat OpenShift GitOps Operator using CLI

You can install Red Hat OpenShift GitOps Operator from the OperatorHub by using the CLI.

## Procedure

1. Create a `openshift-gitops-operator` namespace:

```$ oc create ns openshift-gitops-operator```

2. Create a `OperatorGroup` object YAML file, for example, `gitops-operator-group.yaml`:

```
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-gitops-operator
  namespace: openshift-gitops-operator
spec:
  upgradeStrategy: Default
```

3. Apply the `OperatorGroup` to the cluster:

```$ oc apply -f gitops-operator-group.yaml```

4. Create a `Subscription` object YAML file to subscribe a namespace to the Red Hat OpenShift GitOps Operator, for example, `openshift-gitops-sub.yaml`:

```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-gitops-operator
spec:
  channel: latest
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

5. Apply the `Subscription` to the cluster:

```$ oc apply -f openshift-gitops-sub.yaml```

6. After the installation is complete, verify that all the pods in the `openshift-gitops` namespace are running:

```$ oc get pods -n openshift-gitops```

7. Verify that the pods in the `openshift-gitops-operator` namespace are running:

```$ oc get pods -n openshift-gitops-operator```

# Logging in to the Argo CD instance by using the Argo CD admin account

Use the Argo CD admin account to log in to the default ready-to-use Argo CD instance or the newly installed and deployed Argo CD instance.

*Procedure*

1. In the Administrator perspective of the web console, navigate to **Operators** → **Installed Operators** to verify that the Red Hat OpenShift GitOps Operator is installed.
2. Navigate to the menu → **OpenShift GitOps** → **Cluster Argo CD**. The login page of the Argo CD UI is displayed in a new window.
3. Optional: To log in with your OpenShift Container Platform credentials, ensure you are a user of the `cluster-admins` group and then select the `LOG IN VIA OPENSHIFT` option in the Argo CD user interface.
4. Obtain the password for the Argo CD instance:
  a. Use the navigation panel to go to the **Workloads** → **Secrets** page.
  b. Use the **Project** drop-down list and select the namespace where the Argo CD instance is created.
  c. Select the `<argo_CD_instance_name>-cluster` instance to display the password.
  d. On the **Details** tab, copy the password under **Data → admin.password**.
5. Use `admin` as the **Username** and the copied password as the **Password** to log in to the Argo CD UI in the new window.

## Relevant Documentation

https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.15/html/installing_gitops/installing-openshift-gitops
