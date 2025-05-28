1. Create the directory for installation files
```
$ mkdir /home/{{ INSTALL_USER }}/{{ CLUSTER_NAME }}
```

2. Copy/Create install-config.yaml to installation file directory

Review the following [documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/deploying_installer-provisioned_clusters_on_bare_metal/ipi-install-installation-workflow#configuring-the-install-config-file_ipi-install-installation-workflow) for `install-config.yaml` configuration details.

3. Download required OCP Binaries

A. Create `/home/{{ INSTALL_USER }}/.local/bin/` directory
```
$ cd /home/{{ INSTALL_USER }}/
$ mkdir /.local/bin
```

B. Download OpenShift client

Select the OpenShift client version you want
https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/

Download the selected version’s `openshift-client-linux.tar.gz` file to the bastion host in the `/tmp` directory

C. Extract the client binary

```
$ cd /tmp
$ tar -xvzf openshift-client-linux.tar.gz
```

D. Get release image content

Make note of the release image quay URL to pull from
https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.17.11/release.txt

Ex: 
```
quay.io/openshift-release-dev/ocp-release@sha256:80078b22e5e6e215141bd8300c0e0392ada651334a6f3f4fc340f6a8076d1166
```

E. Create temporary pull secret file with `$HOME/.pull-secret-temp` file

> **_NOTE:_**  For information on creating pull secrets to obtain images from registries, view this [documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.13/html/images/managing-images#using-image-pull-secrets)

```
$ touch $HOME/.pull-secret-temp
$ cp <PULL_SECRET> $HOME/.pull-secret-temp
```

F. Extract `openshift-baremetal` command
```
/tmp/oc adm release extract 
        --command=openshift-baremetal-install 
        --to /tmp/ 
        --from={{ release_image }}        # FROM STEP '3.4'
```

G. Remove temporary pull secret file
```
$ rm $HOME/.pull-secret-temp
```

H. Copy the downloaded binaries from /tmp to the directory created in step 3A
```
$ cp /tmp/oc /home/{{ INSTALL_USER }}/.local/bin/oc-{{ OPENSHIFT_VERSION }}
$ cp /tmp/openshift-baremetal-install /home/{{ INSTALL_USER }}/.local/bin/openshift-baremetal-install-{{ OPENSHIFT_VERSION }}
```

4. Set up KVM on Bastion

A. Install KVM packages (`qemu-kvm`, `libvirt`, `virt-install`)
```
$ sudo apt install qemu-kvm libvirt virt-install
```

B. Enable and start `libvirtd`
```
$ sudo systemctl enable libvirtd
$ sudo systemctl start libvirtd
```

C. Add install user to `libvirt` group
```
$ sudo usermod -a -G libvirt {{ INSTALL_USER }}
```

5. Launch OpenShift Installer
You may now launch the OpenShift installer
