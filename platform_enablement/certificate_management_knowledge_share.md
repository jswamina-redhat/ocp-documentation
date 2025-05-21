# Purpose

This document will provide an introduction to certificate management in OpenShift Container Platform as well as the cert-manager operator.

# Introduction

Certificates are essential for encrypting and securing connections between users, applications, and OpenShift components. OpenShift generates and manages default certificates during installation for internal communication.

Red Hat OpenShift Container Platform (OCP) uses certificates to provide secure connections for the following components:
- masters (API server and controllers)
- etcd
- nodes
- registry
- router

## Purpose of OpenShift Certificates

**Secure Communication:**

Certificates enable encrypted communication between OCP components (ie. API server, etcd server, and web console) and with external clients.

**Authentication and Authorization:**

Certificates help verify the identity of users and applications, ensuring only authorized entities have access to resources.

**Trust Establishment:**

Certificates establish a trusted relationship between OpenShift and its users. This allows for secure access to applications and services.

## Types of OpenShift Certificates

**Default Certificates:**

OpenShift automatically generates and manages a variety of certificates, including those for the API server, etcd server, and web console.

**Custom Certificates:**

Administrators can replace default certificates with their own trusted certificates for specific purposes such as route termination, cluster ingress, or custom application deployments.

## Methods of Certificate Management

**Automatic Management:**

OpenShift provides automated certificate management, including issuance, renewal, and rotation.

**Certificate Expiry Alerts:**

Administrators can set up alerts for certificate expiration to ensure they are aware of when certificates are set to expire. This will help in maintaining continuous security.

**Custom Certificate Integration:**

OpenShift offers flexibility to integrate custom certificates from external Certificate Authorities (CAs) 

## Certificate Use Cases

| Use Case | Explanation |
| ----------- | ----------- |
| API Server | Secures communication between the API server and clients Ex: Developers using the oc command |
| etcd | Ensures secure communication between etcd members and the API server |
| Ingress Controller | Provides HTTPS encryption for applications exposed through OpenShift’s routing |
| Web Console | Protects the web console’s login and admin interface |
| Custom Applications | Allows developers to deploy applications that require secure HTTPS connections |

# Examples

## Replacing the Default Ingress Certificate

You can replace the default ingress certificate in order to establish secure connections to appliances.

This will impact all applications under the `.apps` subdomain. After replacing the certificate, all applications will have encryption provided by the specified certificate.

[Info] Prerequisties:
- You must have a wildcard certificate and its private key, both in the PEM format
- The certificate must have a `subjectAltName` extension of `*.apps.<clustername>.<domain>`

**Procedure:**
1. Create a secret that contains the wildcard certificate and key
```
$ oc create secret tls <certificate> \	[1]
     --cert=</path/to/cert.crt> \		[2]
     --key=</path/to/cert.key> \		[3]
     -n openshift-ingress
```
**EXPLANATION:**

[1] <certificate> is the name of the secret that will contain the certificate and private key

[2] <path/to/cert.crt> is the path to the certificate on your local system

[3] <path/to/cert.key> is the path to the private key associated with this certificate

2. Update the Ingress Controller configuration with the newly created secret
```
$ oc patch ingresscontroller.operator default \
     --type=merge -p \
     '{"spec":{"defaultCertificate": {"name": "<certificate>"}}}' \	[1]
     -n openshift-ingress-operator
```
EXPLANATION:

[1] Replace `<certificate>` with the name used for the secret in the previous step

## Adding API Server Certificates

The default API server certificate is issued by an internal OpenShift Container Platform cluster CA.

Clients outside of the cluster will not be able to verify the API server’s certificate by default. This certificate can be replaced by one that is issued by a CA that clients trust.

You can add additional certificates to the API server to send based on the client’s requested URL, such as when a reverse proxy or load balancer is used.

[Info] Prerequisites
- You must have the certificate and key, in the PEM format, for the client’s URL
- The certificate must be issued for the URL used by the client to reach the API server
- The certificate must have the `subjectAltName` extension for the URL
- If a certificate chain is required to certify the server certificate, then the certificate chani must be appended to the server certificate. Certificate files must be Base64 PEM-encoded and typically have a `.crt` or `.pem` extension
  - Ex: `cat server_cert.pem int2ca_cert.pem int1ca_cert.pem rootca_cert.pem>combined_cert.pem`

When combining certificates, the order of the certificates is important. Each following certificate must directly certify the certificate preceding it, for example:
1. OpenShift Container Platform master host server certificate
2. Intermediate CA certificate that certifies the server certificate
3. Root CA certificate that certifies the intermediate CA certificate

[Warning]
Do not provide a named certificate for the internal load balancer (host name `api-int.<cluster_name>.<base_domain>`)

Doing so will leave your cluster in a degraded state

**Procedure:**
1. Create a secret that contains the certificate and key in the openshift-config namespace
```
$ oc create secret tls <certificate> \	[1]
     --cert=</path/to/cert.crt> \		[2]
     --key=</path/to/cert.key> \		[3]
     -n openshift-config
```
**EXPLANATION:**

[1] <certificate> is the name of the secret that will contain the certificate

[2] <path/to/cert.crt> is the path to the certificate on your local file system

[3] <path/to/cert.key> is the path to the private key associated with this certificate

2. Update the API server to reference the created secret
```
$ oc patch apiserver cluster \
     --type=merge -p \
     '{"spec":{"servingCerts": {"namedCertificates":
     [{"names": ["<hostname>"],					            [1]
     "servingCertificate": {"name": "<certificate>"}}]}}}'	[2]
```
**EXPLANATION:**

[1] Replace `<hostname>` with the hostname that the API server should provide the certificate for

[2] Replace `<certificate>` with the name used for the secret in the previous step

3. Examine the `apiserver/cluster` object and confirm the secret is now referenced
```
$ oc get apiserver cluster -o yaml

## EXAMPLE OUTPUT ##
...
spec:
  servingCerts:
    namedCertificates:
    - names:
      - <hostname>
      servingCertificate:
        name: <certificate>
...
```
## Changing an Application’s Self-Signed Certificate to CA-signed Certificate

Some application templates create a self-signed certificate that is then directly presented by the application to clients.

As an example, by default and as part of the OpenShift Container Platform Ansible installer deployment process, the metrics deployer creates self-signed certificates.

These self-signed certificates are not recognized by browsers. To mitigate this issue, use a publicly signed certificate, then configure it to re-encrypt traffic with the self-signed certificate.

**Procedure:**
1. Delete the existing route
```
$ oc delete route hawkular-metrics -n openshift-infra
```

With the route deleted, the certificates that will be used in the new route with the re-encrypt strategy must be assembled from the existing wildcard and self-signed certificates created by the metrics deployer.

The following certificates must be available:
- Wildcard CA certificate
- Wildcard private key
- Wildcard certificate
- Hawkular CA certificate

Each certificate must be available as a file on the file system for the new route.

Retrieve the Hawkular CA and store it in a file with the following command:
```
$ oc get secrets hawkular-metrics-certs \
    -n openshift-infra \
    -o jsonpath='{.data.ca\.crt}' | base64 \
    -d > hawkular-internal-ca.crt
```

2. Locate the wildcard private key, certificate, and the CA certificate. Place each into a separate file such as `wildcard.key`, `wildcard.crt`, and `wildcard.ca`
```
$ oc create route reencrypt hawkular-metrics-reencrypt \
    -n openshift-infra \
    --hostname hawkular-metrics.ocp.example.com \
    --key wildcard.key \
    --cert wildcard.crt \
    --ca-cert wildcard.ca \
    --service hawkular-metrics \
    --dest-ca-cert hawkular-internal-ca.crt
```

## Cert-manager Operator for Red Hat OpenShift

The cert-manager Operator for Red Hat OpenShift is a cluster-wide service that provides application certificate lifecycle management.

The cert-manager Operator for Red Hat OpenShift allows you to integrate with external certificate authorities and provides certificate provisioning, renewal, and retirement.

The cert-manager project introduces certificate authorities and certificates as resource types in the Kubernetes API, which makes it possible to provide certificates on demand to developers working within your cluster.

The cert-manager Operator provides the following features:
- Support for integrating with external certificate authorities
- Tools to manage certificates
- Ability for developers to self-serve certificates
- Automatic certificate renewal

# Summary
- Certificates allow for encrypting and securing connections between users, applications, and OpenShift components.
- Certificates enable encrypted communication between OCP components, verify the identity of users and applications, and establish a trusted relationship between OpenShift and its users
- You can use default certificates or custom certificates
- The cert-manager Operator for OpenShift provides application certificate lifecycle management

# Conclusion
This was a brief rundown of how certificate management functions in OpenShift Container Platform. Please reach out to your Red Hat resources if you have any questions!

## Relevant Documentation
https://docs.redhat.com/en/documentation/openshift_container_platform/4.1/html/authentication/configuring-certificates#replacing-default-ingress
https://docs.redhat.com/en/documentation/openshift_container_platform/3.11/html/day_two_operations_guide/admin-solutions-certificate-management
https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/security_and_compliance/cert-manager-operator-for-red-hat-openshift
