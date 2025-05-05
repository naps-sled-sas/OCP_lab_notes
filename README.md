# Openshift Notes
> Here are some notes I found useful in setting up my clusters
> - I always install the OCP Virt piece LAST. It will reach out and download around 150+GB of source images for the VM catalog if your StorageClass is configured correctly
> - When setting up local storage for OpenShift to use for VMs the first things I always do is install the 'Local Storage Operator'
> - Once you have that installed you'll need to configure the Local Volume Discovery CR which can be found below
> - Next you install the LVM Operator and configure the LVM Cluster CR which can be found below
> - Notice under `deviceSelector` this is where you'll specify the devices you want to use for the LVM Cluster (You can find this information under the discovered devices from the Local Storage Operator)
## Useful commands

- [Enabling tab completion for Bash](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/cli_tools/openshift-cli-oc#cli-enabling-tab-completion_cli-configuring-cli)
  - `oc completion bash > oc_bash_completion`
  - `sudo cp oc_bash_completion /etc/bash_completion.d/`
- Create Custom Resource (CR) files using the OpenShift CLI:
    - - `oc create -f <file-name>.yaml`
- Watch machine config pools (MCPs) update (Like after setting the Core password)
    - `watch oc get mcp` wait for all to finish upgrading
- [Shutting down the cluster "gracefully"](https://docs.openshift.com/container-platform/4.15/backup_and_restore/graceful-cluster-shutdown.html)
  - Determine the date on which certificates expire and run the following command: `oc -n openshift-kube-apiserver-operator get secret kube-apiserver-to-kubelet-signer -o jsonpath='{.metadata.annotations.auth\.openshift\.io/certificate-not-after}'`
  - Mark all the nodes in the cluster as unschedulable: `for node in $(oc get nodes -o jsonpath='{.items[*].metadata.name}'); do echo ${node} ; oc adm cordon ${node} ; done`
  - Optional? `for node in $(oc get nodes -l node-role.kubernetes.io/worker -o jsonpath='{.items[*].metadata.name}'); do echo ${node} ; oc adm drain ${node} --delete-emptydir-data --ignore-daemonsets=true --timeout=15s --force ; done`
  - Shutdown: `for node in $(oc get nodes -o jsonpath='{.items[*].metadata.name}'); do oc debug node/${node} -- chroot /host shutdown -h 1; done`
- [Set default storageclass](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/storage/index#defining-storage-classes_dynamic-provisioning)
  - Annotations: 
    - storageclass.kubernetes.io/is-default-class: "true"
    - storageclass.kubevirt.io/is-default-virt-class: "true"
- [Approving certificates for newly added nodes to OCP](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/installing_on_any_platform/installing-platform-agnostic#installation-approve-csrs_installing-platform-agnostic)
    - `oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs --no-run-if-empty oc adm certificate approve`
- Configure LVM operator [Persistent Storage using LVM Storage](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/storage/configuring-persistent-storage#persistent-storage-using-lvms)

## [Creating an htpasswd file using Linux (for local logins)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/authentication_and_authorization/configuring-identity-providers#configuring-htpasswd-identity-provider)

- To use the htpasswd identity provider, you must generate a flat file that contains the user names and passwords for your cluster by using htpasswd.
- **Warning this is barely better than using plaintext passwords, so use a throw away password**

### Prerequisites

- Have access to the htpasswd utility. On Red Hat Enterprise Linux this is available by installing the httpd-tools package.

### Procedure

- Create or update your flat file with a username and hashed password:

- `htpasswd -c -B -b </path/to/users.htpasswd> <username> <password>`
- The command generates a hashed version of the password.

### Example output

```Adding password for user user1
Continue to add or update credentials to the file:

$ htpasswd -B -b </path/to/users.htpasswd> <user_name> <password>
```
## Configuring Local Storage
- Install the Local Storage Operator
- Configure the Local Volume Discovery CR
- Install the LVM Operator
- Configure the LVM Cluster CR
> - This example discovers devices based on the hostnames of the servers
```
apiVersion: local.storage.openshift.io/v1alpha1
kind: LocalVolumeDiscovery
metadata:
  generation: 1
  name: auto-discover-devices
  namespace: openshift-local-storage
spec:
  nodeSelector:
    nodeSelectorTerms:
      - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
              - <server1>
              - <server2>
              - <server3>
```
### After installing the LVM Operator 
> - Modify these before using
```
apiVersion: lvm.topolvm.io/v1alpha1
kind: LVMCluster
metadata:
  name: my-lvmcluster
  namespace: openshift-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: 'true'
    storageclass.kubevirt.io/is-default-virt-class: 'true'
spec:
  storage:
    deviceClasses:
    - name: vg1
      fstype: <ext4|xfs>
      default: true
      deviceSelector: 
        paths:
        - /dev/<sda/sdb>
        forceWipeDevicesAndDestroyAllData: <true|false>
      thinPoolConfig:
        name: thin-pool-1
        sizePercent: 90 
        overprovisionRatio: 10
```
- You can also use the disk path to specify EXACT disks to use instead of relying on the /dev/sd* assignments
```
apiVersion: lvm.topolvm.io/v1alpha1
kind: LVMCluster
metadata:
  name: my-lvmcluster
spec:
  storage:
    deviceClasses:
    - name: vg1
      fstype: <ext4|xfs>
      default: true
      deviceSelector: 
        paths:
        - /dev/disk/by-path/pci-0000:87:00.0-nvme-1
        - /dev/disk/by-path/pci-0000:88:00.0-nvme-1
        optionalPaths:
        - /dev/disk/by-path/pci-0000:89:00.0-nvme-1
        - /dev/disk/by-path/pci-0000:90:00.0-nvme-1
        forceWipeDevicesAndDestroyAllData: <true|false>
      thinPoolConfig:
        name: thin-pool-1
        sizePercent: 90 
        overprovisionRatio: 10
```
> - The LVM cluster above may automatically create a StorageClass, which may not be configured correctly for virt. If it does you can delete that SC and use something like the following
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: lvms-vg1
  labels:
    owned-by.topolvm.io/group: lvm.topolvm.io
    owned-by.topolvm.io/kind: LVMCluster
    owned-by.topolvm.io/name: my-lvmcluster
    owned-by.topolvm.io/namespace: openshift-storage
    owned-by.topolvm.io/version: v1alpha1
  annotations:
    description: Provides RWO and RWOP Filesystem & Block volumes
    storageclass.kubernetes.io/is-default-class: 'true'
    storageclass.kubevirt.io/is-default-virt-class: 'true'
provisioner: topolvm.io
parameters:
  csi.storage.k8s.io/fstype: <ext4|xfs>
  topolvm.io/device-class: vg1
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
```
## [Create Core user password for node access from BMC](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/machine_configuration/machine-configs-configure#core-user-password_machine-configs-configure)
> This is NOT recommended, but I find it very helpful on test clusters.
- `mkpasswd -m SHA-512 <'password'>`
- Create a machine config file that contains the core username and the hashed password `oc create -f <file-name>.yaml`: 
```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: <worker or master>
  name: set-core-user-password
spec:
  config:
    ignition:
      version: 3.2.0
    passwd:
      users:
      - name: core 
        passwordHash: <hash from mkpasswd>
```

## Delete PVCs and PVs
- `for pvc in $(oc get pvc -o jsonpath='{.items[*].metadata.name}'); do oc delete persistentvolumeclaims/${pvc}; done`
- `for pv in $(oc get pv -o jsonpath='{.items[*].metadata.name}'); do oc delete persistentvolume/${pv}; done`
