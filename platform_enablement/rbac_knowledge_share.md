# Purpose

This guide provides a basic overview of Role-Based Access Control (RBAC) in OCP. 

# Introduction

RBAC controls **who** can perform **what** actions on **which resources** in your cluster. It uses:
- **Roles**: Collections of permissions (e.g., "view pods" or "create deployments").
- **Bindings**: Link users/groups to roles.
- **Verbs**: Actions like `get`, `list`, `create`, `delete`.
- **Resources**: API objects like pods, services, or deployments.

# Configuring Roles in OpenShift:

OpenShift provides pre-defined roles for common scenarios. 

**Use these before creating custom roles:**

### Project-Level Roles

| Role | Description |
| ----------- | ----------- |
| `admin` | Full control over a project (except quota/limit changes). |
| `edit` | Create/modify resources (pods, services) but not view or modify roles. |
| `view` | Read-only access to most resources. |
| `basic-user` | List projects and access basic project metadata. |

### System-Generated Roles
| Role | Description |
| ----------- | ----------- |
| `cluster-admin` | Superuser access (use sparingly). |

### *View Existing Roles*
```
oc get clusterroles  # For cluster-wide roles
oc get roles -n <project>  # For project-specific roles
```

### Create a Role
- **Local role (project-scoped):**
```
oc create role <role-name> --verb=<verbs> --resource=<resources> -n <project>
```
- **Cluster Role (cluster-wide):**
```
oc create clusterrole <role-name> --verb=<verbs> --resource=<resources>
```

**Example**: Allow viewing pods in the `blue` project:
```
oc create role podview --verb=get --resource=pods -n blue
```

### Usable Verbs in RBAC
Verbs define *what actions* can be performed. Common verbs include:

| Role | Description |  Example Use Case |
| ----------- | ----------- | ----------- |
| `get` | Read individual resources | View pod details |
| `list` | 	List resources | See all pods in a project |
| `watch` | Stream real-time updates | Monitor pod status changes |
| `create` | Create new resources | Deploy an application |
| `update` | Modify existing resources | Scale a deployment |
| `patch` | Partially modify resources | Update environment variables |
| `delete` | Remove resources | Delete a service |
| `use` | Special verb for Security Context Constraints | Allow privilege escalation |

### **Bind the Role to a User:**
```oc adm policy add-role-to-user <role-name> <username> --role-namespace=<project> -n <project>  ```
`# --role-namespace Only needed for local roles`
**Example**: Grant podview role to user2 in the blue project:
```oc adm policy add-role-to-user podview user2 --role-namespace=blue -n blue```

# Configuring Role Groups:
Role Groups are collections of users/service accounts that share permissions. 

They **greatly** simplify RBAC management by letting you assign permissions to multiple users at once.

**Default Groups:**
- `system:authenticated`: All logged-in users
- `system:serviceaccounts`: All service accounts
- `system:unauthenticated`: Anonymous users

### **Commands:**
```
# Create a new group
oc adm groups new dev-team
```
```
# Add users to a group
oc adm groups add-users dev-team user1 user2
```
```
# Bind a role to a group in a project
oc adm policy add-role-to-group edit dev-team -n my-project
```
```
# Remove role from group in a project
oc adm policy remove-role-from-group edit dev-team -n my-project
```

## Best Practices
🚫 Never bind `cluster-admin` unless absolutely necessary. \n
✅ Start with default roles (`view`, `edit`, `admin`) before creating custom roles. \n
✅ Use cluster roles for broad permissions, and local roles for project-specific needs. \n
✅ Avoid too many permissions: Grant only the minimum required access. \n
✅ Use groups instead of individual users for easier management. \n
✅ Audit roles with `oc describe role/<role-name>`. \n
✅ Test permissions with `oc auth can-i`:
```oc auth can-i delete pods --as user2 -n blue```

## Additional Information

https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/authentication_and_authorization/using-rbac#authorization-overview_using-rbac
