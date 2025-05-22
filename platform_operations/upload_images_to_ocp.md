Helpful Utilities

| Syntax | Description | Purpose |
| ----------- | ----------- | ----------- |
| `oc` | openshift cmd line | login to cluster, set default project for virtctl, create resources on cluster |
| `virtctl` | image manager | how images are managed and uploaded to cluster |
| `qemu-img` | image info | can run check and info on qcow imgs to make sure they are not corrupted and querying for size to know how much to allocate |

Requires 4 Steps: (Change name of pvc and Datasource per template, namespace, size)
ex: change all instances of: windows2019-vm, 102G, windows.2k19.virtio

If on windows, manually create yaml file(s) and run oc apply -f filename.yaml

all steps can be done anywhere that can reach the cluster, except for Step 4 - the actual upload – which needs to run somewhere more site local as any big file uploads can congest the wan if executing from outsite the site - recommended site specific utility server - how to get template to the utility server is up to the user (usually NAS sync works, but sometimes it doesn’t)

1. Log into the cluster with oc
```
# can probably be automated by doing some token thing
oc login https://api.nam-par-01.par.nam.gm.com:6443 --web

# using custom asms but would probably be: openshift-virtualization-os-images
# as that is the standard for image template location
oc project openshift-virtualization-os-images
```

2. Create data volume -- storage should be the greater than the image size. can use qemu-img info
   - a small metadata amount (around 1-2 GB). Setting immediate requested annotation allows
ability to start uploading before storage bind is completed in step 2
```
# can use qeum-img utility to check the size of the template:
# qemu-img info <path to template>
#
# but kind of not necessary as current standards are:
#   linux   200G
#   windows 100G
#
# Here we set 102G as some metadata will occupy some of the dv storage
# and if the end size is smaller than the image, it will fail
# It doesn't really matter anyways, portworx will allocate the final size
# and it is usually a lot bigger than necssary for some reason - ex 334G

oc apply -f - <<-EOF
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: windows2019-vm
  namespace: openshift-virtualization-os-images
  annotations:
    cdi.kubevirt.io/storage.bind.immediate.requested: "true"
    cdi.kubevirt.io/storage.usePopulator: "true"
spec:
  source:
    upload: {}
  storage:
    resources:
      requests:
        storage: 102G
    storageClassName: px-rwx-vm
EOF
```
3. Create single use job to trigger pvc allocation as portworx is WaitForFirstConsumer.
Theoretically possible to run data volume creation for all required data volumes, and then mount all necessary volumes to the binder at once. This will just trigger multiple storage allocations
at the same time which should be fine.
```
oc apply  -f - <<-EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: pvc-binder
  namespace: openshift-virtualization-os-images
spec:
  ttlSecondsAfterFinished: 5
  template:
    metadata:
      name: pvc-binder
    spec:
      containers:
      - name: pvc-binder
        image: rhel9
        command: ["echo",  "Hello World"]
        volumeMounts:
          - mountPath: "/mnt"
            name: pvc-volume
      restartPolicy: Never
      volumes:
      - name: pvc-volume
        persistentVolumeClaim:
          claimName: windows2019-vm
EOF
```

4. Upload image to created dv. Make sure to set correct image path and dv name
```
virtctl image-upload dv windows2019-vm --size=102G --image-path=E:\\templates\\win2019feb2025.qcow2 --namespace=openshift-virtualization-os-images
```

5. Create Datasource specifying instance type (instancetype determines whether on it is a bootable volume)
```
# You can actually run this anytime, the datasource will not show up until the pvc
# is created sucessfully, but having it as the final makes the most sense in a
# step by step process.

oc apply -f - <<-EOF
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataSource
metadata:
  name: windows2019-vm
  namespace: openshift-virtualization-os-images
  labels:
    instancetype.kubevirt.io/default-preference: windows.2k19.virtio
spec:
  source:
    pvc:
      name: windows2019-vm
      namespace: openshift-virtualization-os-images
EOF
```

## Combined Template
```
---
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: <DV_NAME>
  namespace: openshift-virtualization-os-images
  annotations:
    cdi.kubevirt.io/storage.bind.immediate.requested: "true"
    cdi.kubevirt.io/storage.usePopulator: "true"
spec:
  source:
    upload: {}
  storage:
    resources:
      requests:
        storage: <SIZE_GB>G
    storageClassName: px-rwx-vm
---
apiVersion: batch/v1
kind: Job
metadata:
  name: pvc-binder-<ID>
  namespace: openshift-virtualization-os-images
spec:
  ttlSecondsAfterFinished: 5
  template:
    metadata:
      name: pvc-binder-<ID>
    spec:
      containers:
      - name: pvc-binder-<ID>
        image: rhel9
        command: ["echo",  "Hello World"]
        volumeMounts:
          - mountPath: "/mnt"
            name: pvc-volume
      restartPolicy: Never
      volumes:
      - name: pvc-volume
        persistentVolumeClaim:
          claimName: <DV_NAME>
---
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataSource
metadata:
  name: <DATASOURCE_NAME>
  namespace: openshift-virtualization-os-images
  labels:
    instancetype.kubevirt.io/default-preference: <PREFERENCE_NAME>
spec:
  source:
    pvc:
      name: <DV_NAME>
      namespace: openshift-virtualization-os-images
```
