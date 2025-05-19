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

# ROSA setup example
> - [Get started with ROSA](https://docs.aws.amazon.com/rosa/latest/userguide/getting-started.html)
You need the AWS CLI AND the ROSA CLI
- [AWS CLI](]https://aws.amazon.com/cli/)
- [ROSA CLI Linux](https://mirror.openshift.com/pub/cgw/rosa/latest/rosa-linux.tar.gz)
- [ROSA CLI Mac](https://mirror.openshift.com/pub/cgw/rosa/latest/rosa-macosx.tar.gz)
- [ROSA CLI Windows](https://mirror.openshift.com/pub/cgw/rosa/latest/rosa-windows.zip)

> Make sure awscli can see your environment variables: 
> - [Configuring environment variables for the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html)
>   - `export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE`
>   - `export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`
>   - `export AWS_DEFAULT_REGION=us-west-2`

## Complete AWS prerequisites
> Make sure your AWS account is set up for ROSA deployment. If you've already set it up, you can continue to the ROSA prerequisites.

- Enable AWS
- Configure Elastic Load Balancer (ELB)
- Set up a VPC for ROSA HCP clusters (optional for ROSA classic clusters)
- Verify your quotas on AWS console

> **cluster-admin password is going to be passed back to you in plain text fyi**
## Example setup

```
[cmcaskil@corvidae ~]$ export AWS_DEFAULT_REGION=us-east-2
[cmcaskil@corvidae ~]$ export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
[cmcaskil@corvidae ~]$ export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
[cmcaskil@corvidae ~]$ rosa create account-roles --mode auto
I: Logged in as 'cmcaskill' on 'https://api.openshift.com'
I: Validating AWS credentials...
I: AWS credentials are valid!
I: Validating AWS quota...
I: AWS quota ok. If cluster installation fails, validate actual AWS resource usage against https://docs.openshift.com/rosa/rosa_getting_started/rosa-required-aws-service-quotas.html
I: Verifying whether OpenShift command-line tool is available...
I: Current OpenShift Client Version: 4.16.0-202504101004.p0.gee354f6.assembly.stream-ee354f6
I: Creating account roles
I: By default, the create account-roles command creates two sets of account roles, one for classic ROSA clusters, and one for Hosted Control Plane clusters.
In order to create a single set, please set one of the following flags: --classic or --hosted-cp
<TRUNCATED>
```
```
[cmcaskil@corvidae ~]$ rosa create cluster
I: Enabling interactive mode
? Cluster name: rosa-test
? Domain prefix (optional):
? Deploy cluster with Hosted Control Plane: No
? Create cluster admin user: Yes
? Create custom password for cluster admin: Yes
? Password: [? for help] ****************
I: cluster admin user is cluster-admin
I: cluster admin password is <password>
? Deploy cluster using AWS STS: Yes
W: In a future release STS will be the default mode.
W: --sts flag won't be necessary if you wish to use STS.
W: --non-sts/--mint-mode flag will be necessary if you do not wish to use STS.
? OpenShift version (default = '4.17.27'): 4.18.11
? Configure the use of IMDSv2 for ec2 instances (default = 'optional'): optional
W: Account roles not created by ROSA CLI cannot be listed, updated, or upgraded.
I: Using arn:aws:iam::649594858475:role/ManagedOpenShift-Installer-Role for the Installer role
I: Using arn:aws:iam::649594858475:role/ManagedOpenShift-Support-Role for the Support role
I: Using arn:aws:iam::649594858475:role/ManagedOpenShift-ControlPlane-Role for the ControlPlane role
I: Using arn:aws:iam::649594858475:role/ManagedOpenShift-Worker-Role for the Worker role
? External ID (optional):
? Operator roles prefix: rosa-test-p9k5
? Deploy cluster using pre registered OIDC Configuration ID: Yes
W: No OIDC Configuration found; will continue with the classic flow.
? Tags (optional):
? Multiple availability zones: No
? AWS region (default = 'us-east-2'): us-east-2
? PrivateLink cluster: No
? Machine CIDR: 10.0.0.0/16
? Service CIDR: 172.30.0.0/16
? Pod CIDR: 10.128.0.0/14
? Install into an existing VPC: Yes
W: No subnets found in current region that are valid for the chosen CIDR ranges
? Continue with default? A new RH Managed VPC will be created for your cluster Yes
? Select availability zones: No
? Enable Customer Managed key: No
? Compute nodes instance type (optional, choose 'Skip' to skip selection. The default value will be provided; default = 'm5.xlarge'): m5.xlarge
? Enable autoscaling: Yes
? Min replicas: 2
? Max replicas: 2
? Configure cluster-autoscaler: Yes
? Balance similar node groups: Yes
? Skip nodes with local storage: Yes
? Log verbosity: 1
? Labels that cluster autoscaler should ignore when considering node group similarity (optional):
? Ignore daemonsets utilization: Yes
? Maximum node provision time: 15m
? Maximum pod grace period: 600
? Pod priority threshold: -10
? Maximum amount of nodes in the cluster: 254
? Minimum number of cores to deploy in cluster: 0
? Maximum number of cores to deploy in cluster: 11520
? Minimum amount of memory, in GiB, in the cluster: 4
? Maximum amount of memory, in GiB, in the cluster: 230400
? Enter the number of GPU limitations you wish to set: 0
? Should scale-down be enabled: No
? How long a node should be unneeded before it is eligible for scale down (optional):
? Node utilization threshold: 0.500000
? How long after scale up should scale down evaluation resume (optional):
? How long after node deletion should scale down evaluation resume (optional):
? How long after node deletion failure should scale down evaluation resume. (optional):
? Worker machine pool labels (optional):
? Host prefix: 23
? Machine pool root disk size (GiB or TiB): 300 GiB
? Enable FIPS support: No
? Encrypt etcd data: No
? Disable Workload monitoring: No
? Customize the default Ingress Controller? No
I: Creating cluster 'rosa-test'
I: To create this cluster again in the future, you can run:
   rosa create cluster --cluster-name rosa-test --sts --cluster-admin-password <password> --role-arn arn:aws:iam::649594858475:role/ManagedOpenShift-Installer-Role --support-role-arn arn:aws:iam::649594858475:role/ManagedOpenShift-Support-Role --controlplane-iam-role arn:aws:iam::649594858475:role/ManagedOpenShift-ControlPlane-Role --worker-iam-role arn:aws:iam::649594858475:role/ManagedOpenShift-Worker-Role --operator-roles-prefix rosa-test-p9k5 --region us-east-2 --version 4.18.11 --ec2-metadata-http-tokens optional --enable-autoscaling --min-replicas 2 --max-replicas 2 --compute-machine-type m5.xlarge --machine-cidr 10.0.0.0/16 --service-cidr 172.30.0.0/16 --pod-cidr 10.128.0.0/14 --host-prefix 23 --autoscaler-balance-similar-node-groups --autoscaler-skip-nodes-with-local-storage --autoscaler-log-verbosity 1 --autoscaler-max-pod-grace-period 600 --autoscaler-pod-priority-threshold -10 --autoscaler-ignore-daemonsets-utilization --autoscaler-max-node-provision-time 15m --autoscaler-max-nodes-total 254 --autoscaler-min-cores 0 --autoscaler-max-cores 11520 --autoscaler-min-memory 4 --autoscaler-max-memory 230400 --autoscaler-scale-down-utilization-threshold 0.500000
I: To view a list of clusters and their status, run 'rosa list clusters'
I: Cluster 'rosa-test' has been created.
I: Once the cluster is installed you will need to add an Identity Provider before you can login into the cluster. See 'rosa create idp --help' for more information.
<TRUNCATED>
I: Run the following commands to continue the cluster creation:

        rosa create operator-roles --cluster rosa-test
        rosa create oidc-provider --cluster rosa-test

I: To determine when your cluster is Ready, run 'rosa describe cluster -c rosa-test'.
I: To watch your cluster installation logs, run 'rosa logs install -c rosa-test --watch'.
[cmcaskil@corvidae ~]$ rosa list ocm-role
I: Fetching ocm roles
I: No ocm roles available
[cmcaskil@corvidae ~]$ rosa create ocm-role --admin
I: Creating ocm role
? Role prefix: ManagedOpenShift
? Permissions boundary ARN (optional):
? Role Path (optional):
? Role creation mode: auto
I: Creating role using 'arn:aws:iam::649594858475:user/open-environment-crwhh-admin'
? Create the 'ManagedOpenShift-OCM-Role-16317861' role? Yes
<TRUNCATED>
I: Linking OCM role
[cmcaskil@corvidae ~]$ rosa create user-role
I: Creating User role
? Role prefix: ManagedOpenShift
? Permissions boundary ARN (optional):
? Role Path (optional):
? Role creation mode: auto
I: Created role 'ManagedOpenShift-User-cmcaskill-Role' with ARN 'arn:aws:iam::649594858475:role/ManagedOpenShift-User-cmcaskill-Role'
I: Linking User role
[cmcaskil@corvidae ~]$
```

## Add identity provider so you can log in