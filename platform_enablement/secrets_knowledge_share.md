# Introduction

Secrets securely store sensitive data like passwords, tokens, or configurations while decoupling credentials from application code.

# Secret Types and Creation

You **must** create a secret before creating the pods that depend on that secret. The value in the `type` field indicates the structure of the secret’s key names and values.
- Opaque
- Docker CFG
- Basic Auth
- SSH Auth
- TLS

### Opaque (`type=opaque`)
- Default secret type for **arbitrary** key-value pairs
- Use When:
  - Storing database credentials
  - API keys for external services
  - Configuration values that shouldn't be hard coded

### **Creation with YAML**
```
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
  namespace: my-namespace
type: Opaque
data:
  username: dmFsdWUtMQ0K
  password: dmFsdWUtMg0KDQo=
stringData:
  hostname: myapp.mydomain.com
```
### **Creation with Literal values:**
```
oc create secret generic db-creds \
  --from-literal=username=dmFsdWUtMQ0K \
  --from-literal=password='dmFsdWUtMg0KDQo='
```

### **Docker CFG (`type=kubernetes.io/dockerconfigjson`)**
- Uses the `.dockercfg` file for required credentials.
- Use When:
  - Pulling images from private registries
  - Pushing builds to authenticated repositories

**Creation with YAML**
```
apiVersion: v1
kind: Secret
metadata:
  name: aregistrykey
  namespace: myapps
type: kubernetes.io/dockerconfigjson 
data:
  .dockerconfigjson
```

### Basic Authentication (`type=kubernetes.io/basic-auth`)

 Create the secret first before using the user name and password to access the private repository:

 ```
$ oc create secret generic <secret_name> \
    --from-literal=username=<user_name> \
    --from-literal=password=<password> \
    --type=kubernetes.io/basic-auth
```
To create a basic authentication secret with a token:
```
$ oc create secret generic <secret_name> \
    --from-literal=password=<token> \
    --type=kubernetes.io/basic-auth
```
### SSH Authentication (type=kubernetes.io/ssh-auth)

 Before using the SSH key to access the private repository, create the secret first:

```
$ oc create secret generic <secret_name> \
    --from-file=ssh-privatekey=<path/to/ssh/private/key> \
    --from-file=<path/to/known_hosts> \ 1
    --type=kubernetes.io/ssh-auth
```

### TLS (`type=kubernetes.io/tls`)

Create a secret with a CA certificate file (recommended).

If your CA uses Intermediate Certificate Authorities, combine the certificates for all CAs in a ca.crt file. Run the following command:

```
$ cat intermediateCA.crt intermediateCA.crt rootCA.crt > ca.crt
```

Create the secret:

```
$ oc create secret generic mycert --from-file=ca.crt=</path/to/file>
```

> **_NOTE:_**  You must use the key name ca.crt.**

Alternatives

Consider these instead of secrets if:
- ConfigMaps: You have on-sensitive configuration data
- Vault Integration: You need dynamic secret generation/rotation

### More Information
https://docs.redhat.com/en/documentation/openshift_container_platform/3.11/html/developer_guide/dev-guide-secrets#dev-guide-secrets-using-secrets
