# Introduction

GroupSync automatically mirrors your organization's existing LDAP or AD groups into OCP, keeping user access permissions in sync with your central directory.

[Prerequisites]
- Cluster-admin privileges
- LDAP server details (host, bind DN, password)
- CA certificate for secure connection
- Base DN for user/group searches

## Creating an RFC2307 Sync Config File

The RFC 2307 schema requires you to provide an LDAP query definition for both user and group
entries, as well as the attributes with which to represent them in the internal OpenShift Container
Platform records.

```
apiVersion: v1
kind: LDAPSyncConfig
insecure: false
url: ldaps://ldap.example.com:636
bindDN: uid=admin,cn=users,dc=example,dc=com
bindPassword:
  file: "/etc/secrets/bindPassword"
rfc2307:
  groupsQuery:
    baseDN: "ou=groups,dc=example,dc=com"
    scope: sub
    derefAliases: never
    pageSize: 0
groupUIDAttribute: dn
groupNameAttributes: [ cn ]
groupMembershipAttributes: [ member ]
usersQuery:
  baseDN: "ou=users,dc=example,dc=com"
  scope: sub
  derefAliases: never
  pageSize: 0
userUIDAttribute: dn
userNameAttributes: [ uid]
tolerateMemberNotFoundErrors: false
tolerateMemberOutOfScopeErrors: false
```
[Warning] This config only works if you have created a cron job as described below in Auto Syncing LDAP Groups 
[Info] RFC2307 doesn’t support nested groups natively.

## Syncing LDAP Groups

[Note] If you’d like to dry run these commands to see their results, omit the `--confirm` flag.

To sync all groups from the LDAP server with OCP you can run

```
$ oc adm groups sync --sync-config=config.yaml --confirm
```

## Auto Syncing LDAP Groups

To do this we configure a cron job. This assumes you have a secret named `ldap-secret` and config map named `ca-config-map` from when you initially configured LDAP.

In the project where the cron job will run, copy the `ldap-secret` and `ca-config-map` from `openshift-config`

The cron job will need a service account, cluster role, cluster role binding, a config map, and then the cron job yaml

### Define service account YAML

```
kind: ServiceAccount
apiVersion: v1
metadata:
  name: ldap-group-syncer
  namespace: ldap-sync
```
Then create the account:
```
$ oc create -f ldap-sync-service-account.yaml
```

### Define cluster role YAML
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ldap-group-syncer
rules:
  - apiGroups:
      - user.openshift.io
    resources:
      - groups
    verbs:
      - get
      - list
      - create
      - update
```

Then create the cluster role:
```
$ oc create -f ldap-sync-cluster-role.yaml
```

### Define role binding YAML
```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ldap-group-syncer
subjects:
  - kind: ServiceAccount
    name: ldap-group-syncer              
    namespace: ldap-sync
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ldap-group-syncer
```
Then create the binding:

```
$ oc create -f ldap-sync-cluster-role-binding.yaml
```

### Define sync configuration config map YAML

[Note] The `sync.yaml` field of this config map is just your LDAP sync config file from above.
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: ldap-group-syncer
  namespace: ldap-sync
data:
  sync.yaml: |                                 
    kind: LDAPSyncConfig
    apiVersion: v1
    url: ldaps://10.0.0.0:389                  
    insecure: false
    bindDN: cn=admin,dc=example,dc=com         
    bindPassword:
      file: "/etc/secrets/bindPassword"
    ca: /etc/ldap-ca/ca.crt
    rfc2307:                                   
      groupsQuery:
        baseDN: "ou=groups,dc=example,dc=com"  
        scope: sub
        filter: "(objectClass=groupOfMembers)"
        derefAliases: never
        pageSize: 0
      groupUIDAttribute: dn
      groupNameAttributes: [ cn ]
      groupMembershipAttributes: [ member ]
      usersQuery:
        baseDN: "ou=users,dc=example,dc=com"   
        scope: sub
        derefAliases: never
        pageSize: 0
      userUIDAttribute: dn
      userNameAttributes: [ uid ]
      tolerateMemberNotFoundErrors: false
      tolerateMemberOutOfScopeErrors: false
```
Then create the config map:
```
$ oc create -f ldap-sync-config-map.yaml
```

### Define the cron job YAML
```
kind: CronJob
apiVersion: batch/v1
metadata:
  name: ldap-group-syncer
  namespace: ldap-sync
spec:                                                                                
  schedule: "*/30 * * * *"                                                           
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      backoffLimit: 0
      ttlSecondsAfterFinished: 1800                                                  
      template:
        spec:
          containers:
            - name: ldap-group-sync
              image: "registry.redhat.io/openshift4/ose-cli:latest"
              command:
                - "/bin/bash"
                - "-c"
                - "oc adm groups sync --sync-config=/etc/config/sync.yaml --confirm" 
              volumeMounts:
                - mountPath: "/etc/config"
                  name: "ldap-sync-volume"
                - mountPath: "/etc/secrets"
                  name: "ldap-bind-password"
                - mountPath: "/etc/ldap-ca"
                  name: "ldap-ca"
          volumes:
            - name: "ldap-sync-volume"
              configMap:
                name: "ldap-group-syncer"
            - name: "ldap-bind-password"
              secret:
                secretName: "ldap-secret"                                            
            - name: "ldap-ca"
              configMap:
                name: "ca-config-map"                                                
          restartPolicy: "Never"
          terminationGracePeriodSeconds: 30
          activeDeadlineSeconds: 500
          dnsPolicy: "ClusterFirst"
          serviceAccountName: "ldap-group-syncer"
```
The LDAP sync command gets passed the sync configuration file defined in the config map, as well as the `ldap-secret` and `ca-config-map` created when LDAP was configured.

Then create the cron job:
```
oc create -f ldap-sync-cron-job.yaml
```

## Pruning LDAP Groups
You can also remove groups from OCP if the records on the LDAP server are no longer present. It accepts the same sync config file you used to synchronize groups.
```
oc adm prune groups --sync-config=config.yaml --confirm
```

## Verification and Troubleshooting
**List synced groups**
```
oc get groups
```

**Check group details**
```
oc describe group ocp-admins
```

**Check your sync results**
```
oc get groups -l ldap-sync/true
```

**Inspect sync job logs**
```
oc logs -f jobs/<ldap-group-sync-pod>
```

## Relevant Documentation
https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/authentication_and_authorization/ldap-syncing#ldap-syncing
https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html-single/nodes/index#nodes-nodes-jobs-creating-cron_nodes-nodes-jobs
