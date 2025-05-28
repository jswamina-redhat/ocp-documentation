# Purpose

This document should provide an introduction to `etcd` backups in OpenShift Container Platform and information on how to back up `etcd` data.

# Introduction

`etcd` is a strongly consistent, distributed key-value store that provides a reliable way to store data that needs to be accessed by a distributed system or cluster of machines.

Key features of `etcd`:
| Feature | Explanation |
| ----------- | ----------- |
| Key-Value Store | Data is stored in a key-value format, where each key is associated with a corresponding value |
| Distributed System | Designed to operate across multiple nodes,mensuring data is accessible even if individual nodes fail |
| High Availability | `etcd`'s distributed nature allows it to continue operating even if some nodes fail, ensuring high availability |
| Kubernetes Backing Store | In Kubernetes, `etcd` stores all cluster-critical data, including configuration settings, deployments, and other resource definitions |

`etcd` is a reliable and consistent key-value store that provides a foundation for managing data in Kubernetes clusters.

# Backing Up `etcd` Data
Red Hat recommends backing up your cluster’s `etcd` data regularly and storing it in a secure location outside of the OpenShift environment. 


> **_INFO:_** Do not take an `etcd` backup prior to the first rotation of certificates (24 hours after initial install). Doing so will cause the backup to contain expired certificates.

> **_INFO:_**  Taking `etcd` backups is a blocking action. Therefore it is recommended to take backups outside of peak usage hours. It is also important to take an `etcd` backup before cluster upgrades.

> **_WARNING:_**  Back up your cluster’s etcd data by performing a single invocation of the backup script on a master host. Do not take a backup for each master host. Only save a backup from a single control plane host. Do not take a backup from each control plane host in the cluster.

**Prerequisites**:
- You have access to the cluster as a user with the cluster-admin role
- You have checked whether the cluster-wide proxy is enabled

> **_NOTE:_**  You can check whether the proxy is enabled by reviewing the output of `oc get proxy cluster -o yaml`. The proxy is enabled if the `httpProxy`, `httpsProxy`, and `noProxy` fields have values set.

**Procedure:**
1. Start a debug session
```
$ oc debug --as-root node/<node_name>
```

2. Change your root director to `/host` int he debug shell
```
sh-4.4# chroot /host
```

3. f the cluster-wide proxy is enabled, export the `NO_PROXY`, `HTTP_PROXY`, and `HTTPS_PROXY` environment variables by running the following commands
```
$ export HTTP_PROXY=http://dcproxy.gm.com:8080
$ export HTTPS_PROXY=http://dcproxy.gm.com:8080
$ export NO_PROXY=.gm.com,localhost,127.0.0.1,10.0.0.0/8,172.0.0.0/8,198.0.0.0/8
```

4. Run the `cluster-backup.sh` script in the debug shell and pass in the location to save the backup to

> **_INFO:_**  The `cluster-backup.sh` script is maintained as a component of the `etcd` Cluster Operator and is a wrapper around the `etcdctl snapshot save` command
```
sh-4.4# /usr/local/bin/cluster-backup.sh /home/core/assets/backup
```
Example Output:
```
found latest kube-apiserver: /etc/kubernetes/static-pod-resources/kube-apiserver-pod-6
found latest kube-controller-manager: /etc/kubernetes/static-pod-resources/kube-controller-manager-pod-7
found latest kube-scheduler: /etc/kubernetes/static-pod-resources/kube-scheduler-pod-6
found latest etcd: /etc/kubernetes/static-pod-resources/etcd-pod-3
ede95fe6b88b87ba86a03c15e669fb4aa5bf0991c180d3c6895ce72eaade54a1
etcdctl version: 3.4.14
API version: 3.4
{"level":"info","ts":1624647639.0188997,"caller":"snapshot/v3_snapshot.go:119","msg":"created temporary db file","path":"/home/core/assets/backup/snapshot_2021-06-25_190035.db.part"}
{"level":"info","ts":"2021-06-25T19:00:39.030Z","caller":"clientv3/maintenance.go:200","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":1624647639.0301006,"caller":"snapshot/v3_snapshot.go:127","msg":"fetching snapshot","endpoint":"https://10.0.0.5:2379"}
{"level":"info","ts":"2021-06-25T19:00:40.215Z","caller":"clientv3/maintenance.go:208","msg":"completed snapshot read; closing"}
{"level":"info","ts":1624647640.6032252,"caller":"snapshot/v3_snapshot.go:142","msg":"fetched snapshot","endpoint":"https://10.0.0.5:2379","size":"114 MB","took":1.584090459}
{"level":"info","ts":1624647640.6047094,"caller":"snapshot/v3_snapshot.go:152","msg":"saved","path":"/home/core/assets/backup/snapshot_2021-06-25_190035.db"}
Snapshot saved at /home/core/assets/backup/snapshot_2021-06-25_190035.db
{"hash":3866667823,"revision":31407,"totalKey":12828,"totalSize":114446336}
snapshot db and kube resources are successfully saved to /home/core/assets/backup
```

In this example, two files are created in the `/home/core/assets/backup/` directory on the control plane host:
- `snapshot_<datetimestamp>.db`: This file is the etcd snapshot. The `cluster-backup.sh` script confirms its validity
- `static_kuberesources_<datetimestamp>.tar.gz`: This file contains the resources for the static pods. If `etcd` encryption is enabled, it also contains the encryption keys for the etcd snapshot

> **_INFO:_**  If `etcd` encryption is enabled, it is recommended to store this second file separately from the `etcd` snapshot for security reasons. However, this file is required to restore from the `etcd` snapshot.

Keep in mind that `etcd` encryption only encrypts values, not keys. This means that resource types, namespaces, and object names are unencrypted.

# Restore a Previous Cluster State
You may need to restore a previous cluster state due to some issues. This is why we keep etcd backups on hand by creating a snapshot to restore the cluster state.

This can be used to recover from the following situations:
- The cluster has lost the majority of control plane hosts (quorum loss)
- An administrator has deleted something critical and must restore to recover the cluster

> **_WARNING:_**  Restoring to a previous cluster state is a destructive and destabilizing action to take on a running cluster. This should only be used as a last resort.

If you are able to retrieve data using the Kubernetes API server, then `etcd` is available and you should not restore using an etcd backup.

Restoring a previous cluster state depends on how the cluster was initially created (IPI vs UPI) along with some additional scenarios. Please refer to this documentation to find the procedure relevant to you.

https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/backup_and_restore/control-plane-backup-and-restore#dr-restoring-cluster-state

# Summary
- `etcd` provides a foundation for managing data in Kubernetes clusters
- `etcd`’s key features are:
  - Key-Value Store
  - Distributed System
  - High Availability
  - Kubernetes Backing Store
- Back up your cluster’s `etcd` data regularly and store it in a secure location outside of the OpenShift environment

# Conclusion
You should now understand what `etcd` backups are and how to backup your `etcd` data should you need it for a cluster restoration.

## Relevant Documentation
https://etcd.io/
https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/backup_and_restore/control-plane-backup-and-restore#backup-etcd

https://gm-sdv.atlassian.net/wiki/spaces/IT/pages/429293626/High+Level+OCP+Cluster+Provisioning#Day-2-Configuration-(RH)
