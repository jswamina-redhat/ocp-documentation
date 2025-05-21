# Purpose

The following is a basic breakdown of how cluster monitoring and alerting configuration occurs manually. 

This document is intended to provide a foundation of knowledge for related automation tasks.

# Introduction

There are two main files in which monitoring components are configured:
- `cluster-monitoring-config.yaml`
- `user-workload-monitoring-config.yaml`

In order to create the cluster monitoring config map, create the following `yaml` file:
`cluster-monitoring-config.yaml`
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
```
Save this file and apply it to the cluster: `oc apply -f cluster-monitoring-config.yaml`

In order to configure core OpenShift Container Platform moniitoring components you must have cluster access with a user that has the `cluster-admin` cluster role.


To create the user-defined workload monitoring config map, create the following `yaml` file:
`user-workload-monitoring-config.yaml`
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-workload-monitoring-config
  namespace: openshift-user-workload-monitoring
data:
  config.yaml: |
```
Apply it to the cluster: `oc apply -f user-workload-monitoring-config.yaml`

After creation, you can edit the configmaps from the command line with this command:

`oc -n <namespace> edit configmap <config-map-name>`

## Configuration

When you add or edit different configurations within these files, you are configuring components of the monitoring stack.

In order to configure these files after they have been created, you’ll need the `cluster-admin` role to configure core OpenShift Container Platform monitoring components (i.e. the `cluster-monitoring-config` configmap).

And for user-workload monitoring (`user-workload-monitoring-config` configmap), you’ll need the `user-workload-monitoring-config-edit` role in the `openshift-user-workload-monitoring` project (if you are a user that is not a `cluster-admin`).

All configurations will be within the `config.yaml |` section of each of these configmaps as a `<key>: <value>` pair.
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    <component>:
      <configuration_for_the_component>
```
You can find the configuration reference for all of the components here: https://docs.openshift.com/container-platform/4.15/observability/monitoring/config-map-reference-for-the-cluster-monitoring-operator.html#clustermonitoringconfiguration

# Common Configuration Scenarios

### Enable Monitoring for User-Defined Projects
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
```

### Prometheus

Let’s say you want to build on the previous configuration and configure a persistent volume claim (PVC) for your Prometheus instance that monitors core OpenShift Container Platform components.

It would look something like this:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
    prometheusK8s: 
      volumeClaimTemplate:
        spec:
          storageClassName: fast
          volumeMode: Filesystem
          resources:
            requests:
              storage: 40Gi
```
Make sure you save the file and exit to automatically apply the changes. 

In this example, `prometheusK8s` is the monitoring component being configured.

Maybe you’d like to configure a Prometheus instance that monitors user-defined projects only.

You can edit the `user-workload-monitoring-config` configmap like so:

(Remember: to edit a created configmap, use the following command: `oc -n <namespace> edit configmap <config-map-name>`)
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-workload-monitoring-config
  namespace: openshift-user-workload-monitoring
data:
  config.yaml: |
    prometheus: 
      retention: 24h 
      resources:
        requests:
          cpu: 200m 
          memory: 2Gi
```
Here, `prometheus` is the component being configured. Don’t forget to save the file and exit the editor to apply the changes.

### Alertmanager

Users can be allowed to create user-defined alert routing configurations that use the main platform instance of alertmanager.
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    # ...
    alertmanagerMain:
      enableUserAlertmanagerConfig: true 
    # ...
```
The `alertmanagerMain` property defines settings for the Alertmanager component in the `openshift-monitoring` namespace.

Setting its `enableUserAlertmanagerConfig` property to `true` allows users to create user-defined alert routing configurations that use the main platform instance.

# Summary

The two configmaps for configuring the monitoring stack are:
- `cluster-monitoring-config`
- `user-workload-monitoring-config`

To edit configmaps you run the following command on the command-line:
`oc -n <namespace> edit configmap <config-map-name>`

After editing these configmaps that are related to the monitoring stack, the information gets sent to the Cluster Monitoring Operator (CMO) which then configures the core components of the monitoring stack.

# Conclusion

Hopefully you now have a better understanding of the building blocks when it comes to configuring cluster monitoring.

Ultimately, our goal will be to automate this setup (i.e. creating the configmaps with their necessary configurations) via AAP or GitOps.

Having the background knowledge of how it works should help facilitate debugging when it comes to the automation portion.

Feel free to reach out to any of the Red Hat resources if you have any questions or would like further clarification.

## Relevant Documentation
https://docs.openshift.com/container-platform/4.16/observability/monitoring/configuring-the-monitoring-stack.html#creating-cluster-monitoring-configmap_configuring-the-monitoring-stack
https://docs.openshift.com/container-platform/4.15/observability/monitoring/config-map-reference-for-the-cluster-monitoring-operator.html#clustermonitoringconfiguration
https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/monitoring/managing-alerts#managing-alerts-as-an-administrator
https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/postinstallation_configuration/configuring-alert-notifications
