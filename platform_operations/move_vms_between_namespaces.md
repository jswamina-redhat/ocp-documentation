# Purpose
This document will provide steps for moving VMs between namespaces in OpenShift Container Platform.

# Introduction
In Kubernetes, it is not possible to move objects between namespaces or rename existing namespaces. To "move" a virtual machine from one namespace to another we can migrate it using MTV. The migration will create a new virtual machine object in the target namespace. It will also copy all the VM disks.

MTV can migrate virtual machines between two different OpenShift clusters. We can create a migration plan where the source and target provider are the same OpenShift cluster. Just the source and target namespaces will differ.

There is one issue connected with such a migration plan. OpenShift checks that the network interface MAC addresses are unique across all VMs in the cluster. Creating a second VM with a MAC address already in use is not possible. To work around this issue, we will remove the network interfaces prior to the VM migration and add them back after the migration is completed. This workaround imposes a downtime on the VM while migrating.

A machine can be "moved" between namespaces in several steps. In the following examples, replace the `sourcens` with the name of the current VM namespace, `destns` with the name of the namespace the VM should be moved to. Replace the `example` with the name of your VM.

First, shut down the virtual machine.

Make a backup of the virtual machine definition:

```
$ oc get -n sourcens vm example -o yaml > example.backup.yaml
```

Save the VM's network interface configuration into a separate JSON file:
```
$ oc get -n sourcens vm example -o json | jq '{spec:{template:{spec:{domain:{devices:{interfaces:(.spec.template.spec.domain.devices.interfaces)}},networks:(.spec.template.spec.networks)}}}}' > example.json
```

Example JSON file content:
```
{
  "spec": {
    "template": {
      "spec": {
        "domain": {
          "devices": {
            "interfaces": [
              {
                "macAddress": "02:cb:4d:00:00:02",
                "masquerade": {},
                "model": "virtio",
                "name": "default"
              }
            ]
          }
        },
        "networks": [
          {
            "name": "default",
            "pod": {}
          }
        ]
      }
    }
  }
}
```

Remove the network interfaces from the VM. Before issuing the following command, make sure that you backed up the VM definition as instructed previously:
```
$ oc patch vm -n sourcens example --type merge --patch '{"spec":{"template":{"spec":{"domain":{"devices":{"interfaces":[]}},"networks":[]}}}}'
```

Now you can use MTV to migrate the VM from one namespace to another.

After the migration is complete, you can restore the VM's network interface configuration from the JSON file we saved previously:

```
$ oc patch vm -n destns example --type merge --patch "$(cat example.json)"
```
You can now start the VM and check that it works as expected.

As a last step, delete the original VM.


