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
