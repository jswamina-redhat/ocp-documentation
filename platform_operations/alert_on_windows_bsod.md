This section describes OpenShift configuration that detects the Windows Blue Screen of Death (BSOD) and fires a Prometheus alert. The configuration includes two parts:

1. Enable readiness probe for the virtual machine(s)
2. Create a Prometheus alerting rule that fires if a virtual machine is not ready

# Enable Readiness Probe for VM

The readiness probe periodically pings the QEMU guest agent that is deployed inside of the virtual machine. If the virtual machine crashes (BSOD), the guest agent becomes unreachable and the probe starts failing. The status of the virtual machine pod turns to not ready.

The readiness probe needs to be configured separately for each virtual machine. The following commands enable the readiness probe for a single virtual machine.

First, create a patch `patch.yaml` file with the probe definition:
```
spec:
  template:
    spec:
      readinessProbe:
        guestAgentPing: {}
        initialDelaySeconds: 120
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 3
        successThreshold: 3
```
Use the above file to patch a virtual machine. Before issuing the patch command, replace the namespace and name of the virtual machine:
```
$ oc patch vm --type merge --patch-file patch.yaml -n <vm_namespace> <vm_name>
```

Check that the patch was applied correctly by listing the current readiness probe configuration.  Before issuing the patch command, replace the namespace and name of the virtual machine:
```
$ oc get vm -o jsonpath='{.spec.template.spec.readinessProbe}' -n <vm_namespace> <vm_name>
```

Example command output:
```
{"failureThreshold":3,"guestAgentPing":{},"initialDelaySeconds":120,"periodSeconds":10,"successThreshold":3,"timeoutSeconds":5}
```

After configuring the readiness probe, you must restart the VM for the configuration to take effect. You can restart the VM using the OpenShift Web Console or the virtctl command. Before issuing the virtctl command, replace the namespace and name of the virtual machine:
```
$ virtctl restart -n <vm_namespace> <vm_name>
```

Note that when restarting the VM with the readiness probe configured, it takes some time (up to three minutes) for the VM to become ready. To watch the VM’s state, you can issue the command:
```
$ oc get po
NAME                                          READY   STATUS    RESTARTS   AGE
...
virt-launcher-windows2016-w2qmf               0/1     Running   0          101s
...
```

After about three minutes, the VM becomes ready:
```
$ oc get po
NAME                                          READY   STATUS    RESTARTS   AGE
...
virt-launcher-windows2016-w2qmf               1/1     Running   0          2m51s
...
```

You can also verify, that the VM pod configuration now includes the definition of the readiness probe. Before issuing the following command, replace the name of the virtual machine pod with your pod’s name:
```
$ oc get po virt-launcher-windows2016-w2qmf -o yaml
```
```
...
    readinessProbe:
      exec:
        command:
        - virt-probe
        - --domainName
        - kubevirt-example_example
        - --timeoutSeconds
        - "5"
        - --guestAgentPing
...
```

## Testing Readiness Probe

You can test the readiness probe by manually triggering the Windows BSOD in your virtual machine. 

Log into your Windows virtual machine and disable automatic restart on blue screen of death (BSOD) by running this PowerShell command:
```
$ wmic recoveros set AutoReboot = False
```

Reboot the virtual machine for the above change to take effect. After the reboot, log into the virtual machine again and trigger the BSOD by running the following command in PowerShell:
```
$ taskkill /IM svchost.exe /f
```

After issuing the above command, you should now see the Windows BSOD

Less than a minute later, the virtual machine pod status turns to not ready:
```
$ oc get po
NAME                                          READY   STATUS    RESTARTS   AGE
...
virt-launcher-windows2016-w2qmf               0/1     Running   0          21m
...
```

# Creating Alerting Rule

In this section, we will create a Prometheus alerting rule. This rule needs to be created once per OpenShift cluster.

First, save the rule definition below into a file named `virtual-machine-pod-not-ready-alertingrule.yaml`:

```
apiVersion: monitoring.openshift.io/v1
kind: AlertingRule
metadata:
  name: virtual-machine-pod-not-ready
  namespace: openshift-monitoring
spec:
  groups:
  - name: vm.rules
    rules:
    - alert: VirtualMachinePodNotReady
      expr: >-
        kube_pod_container_status_ready{pod=~"virt-launcher.*"} == 0
      for: 5m
      annotations:
        summary: Virtual machine pod is not ready
        description: >-
          Virtual machine pod {{ $labels.pod }} in namespace {{ $labels.namespace }} cannot reach ready state. A readiness
          probe is failing. If the readiness probe checks the health of the QEMU agent then the QEMU agent running inside
          the VM is not reporting. The QEMU agent could have crashed or the guest operating system could have crashed (e.g.
          Windows blue screen of death - BSOD).
      labels:
        severity: warning
```

Next, apply the alerting rule to the cluster:
```
$ oc apply -f virtual-machine-pod-not-ready-alertingrule.yaml
```

After the rule has been created and if there is a VM in the cluster that crashed with Windows BSOD, you will receive an alert
