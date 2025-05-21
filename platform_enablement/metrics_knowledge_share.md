# Purpose

This document should provide a baseline understanding of how metrics work in OpenShift Container Platform. 

# Introduction

Metrics are scraped from the `/metrics` endpoint on the basis of `ServiceMonitors`. 

`ServiceMonitors` require a corresponding `Service` to scrape from. A `ServiceMonitor` is used to tell our `Prometheus` instance where our application monitoring endpoint is.

# Procedure

Let’s go over a basic example of setting up a ServiceMonitor to scrape metrics from an example application.

## Deployment & Service

First, we need to set up our example app `Deployment`:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: prometheus-example-app
  name: prometheus-example-app
  namespace: ns1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-example-app
  template:
    metadata:
      labels:
        app: prometheus-example-app
    spec:
      containers:
      - image: ghcr.io/rhobs/prometheus-example-app:0.4.2
        imagePullPolicy: IfNotPresent
        name: prometheus-example-app
```
To deploy: `oc apply -f <file_name>.yaml`

Now let’s create the `Service` that will expose this application’s metrics:
```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: prometheus-example-app
  name: prometheus-example-app
  namespace: ns1
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
    name: web
  selector:
    app: prometheus-example-app
  type: ClusterIP
```
Apply the `Service`: `oc apply -f <file_name>.yaml`

There are a few things to note here:
- The `Service` has the label `app: prometheus-example-app` which is the label the `ServiceMonitor` will look for

## ServiceMonitor

This is the `ServiceMonitor` we would create for our example application:
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: prometheus-example-monitor
  namespace: ns1
spec:
  endpoints:
  - interval: 30s  			                      # decreasing this increases resource usage 
    port: web
    scheme: http
  selector:
    matchLabels:
      app: prometheus-example-app
```
Apply the `ServiceMonitor` manifest: `oc apply -f <file_name>.yaml`

There are a three pieces of information we need to verify to know we have correctly configured the `ServiceMonitor`:
- Verify the `Service` is deployed in the namespace specified in the `ServiceMonitor` (`ns1` in our case)
- Verify the ports match (`web` in this example)
- The `selector` in the `ServiceMonitor` defines the label that the `ServiceMonitor` will look for. Make sure this matches what is in the `Service`
```
// Service
...
metadata:
  labels:
    app: prometheus-example-app
...
```
```
// ServiceMonitor
...
spec:
...
  selector:
    matchLabels:
      app: prometheus-example-app
```

# Summary
- Metrics are scraped via a `ServiceMonitor` which has a corresponding `Service` that exposes traffic to our deployment
- To verify correct `ServiceMonitor` configuration:
  - Check that the `Service` is deployed in the namespace that is specified in the `ServiceMonitor`
  - Check that the ports in the `Service` and `ServiceMonitor` match
  - Make sure the `ServiceMonitor` has the correct `selector` labels

# Conclusion
This was a basic rundown on how metrics are scraped in OpenShift Container Platform. This should assist in understanding how these different resources work together and how to automate tasks in the future.

## Relevant Documentation

https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/monitoring/managing-metrics#about-querying-metrics_managing-metrics
https://prometheus.io/docs/introduction/overview/
https://prometheus.io/docs/concepts/metric_types/
https://prometheus.io/docs/prometheus/latest/querying/basics/
