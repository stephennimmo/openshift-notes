# OpenShift Virtualization

## Prerequisites

* Cluster worker nodes have been installed on bare metal
* Hardware [requirements](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/installing#virt-hardware-os-requirements_preparing-cluster-for-virt) have been met
* Storage has been [configured](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/installing#storage-requirements_preparing-cluster-for-virt) for the cluster
    * A default OpenShift Virtualization or OpenShift Container Platform storage class has been created
    * To mark a storage class as the default for virtualization workloads, set the annotation `storageclass.kubevirt.io/is-default-virt-class` to `true`
    * Don't forget to [read](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/installing#virt-about-storage-volumes-for-vm-disks_preparing-cluster-for-virt) about the need for RWX
* NMState Operator has been installed
    * All [networking setup](../networking/index.md) in terms of underlay networks have been created
    * Optional, but recommended, [dedicated network for live migration](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/networking#virt-dedicated-network-live-migration)

## Install

Install the [OpenShift Virtualization Operator](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/virtualization/installing#virt-installing-virt-operator_installing-virt) via the web console.

## Optional Post-Install

* [Enable default access to VM guest system logs](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/virtualization/index#virt-enable-guest-log-default-cli_virt-troubleshooting)

## Example cloud-init for Static IP

```yaml
networkData: |
  version: 2
  ethernets:
    eth1:
      dhcp4: no
      addresses:
        - 10.37.0.50/24
      gateway4: 10.37.0.1
      nameservers:
        addresses:
          - 10.3.0.3
          - 9.9.9.9
      dhcp6: no
      accept-ra: false
userData: |
  #cloud-config
  user: cloud-user
  password: Pass123!
  chpasswd:
    expire: false
```

## Descheduler for Live Migration

If you are running OpenShift Virtualization with the Kube Descheduler Operator, use the `DevPreviewLongLifecycle` profile. This profile handles long-running VM workloads and triggers live migrations instead of pod deletions when rebalancing.

Each `VirtualMachine` must be annotated to opt in to descheduler evictions:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: example-vm
spec:
  template:
    metadata:
      annotations:
        descheduler.alpha.kubernetes.io/evict: "true"
```

!!! note
    The VM must have a `LiveMigrate` eviction strategy set for the descheduler to trigger a live migration rather than deleting the pod.
