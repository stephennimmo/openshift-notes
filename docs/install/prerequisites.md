# Prerequisites for OpenShift Install

For the installation, the documentation assumes a bare metal environment in an on-premise data center in an enterprise environment. Most importantly, your network, security, and storage teams will need to be involved. If you have ticketing processes for making changes to networks, DNS, and other services, then it will be imparative to get everything planned, submitted and **validated** prior to the installation. 

* Evaluation subscriptions are required for any proof of concept using Red Hat software.
* You need a [Red Hat account](https://www.redhat.com/wapps/ugc/register.html) associated with your organization. Do not use personal Red Hat accounts for business purposes. 
* You will need to open your firewall to whitelist all the Red Hat associated content sites. Documentation for the firewall configuration ([link](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installation_configuration/configuring-firewall)).
* You will need to create two DNS records for each cluster you install. One of the DNS records is a wildcard - `*.apps.<cluster_name>.<base_domain>` ([link](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/installing_on_any_platform/index#installation-dns-user-infra_installing-platform-agnostic)). 
* You will need to be able to download and install an OEM version of Red Hat Enterprise Linux. Download Red Hat Enterprise Linux 9.x Binary DVD ([link](https://access.redhat.com/downloads/content/rhel))

## Required Hardware

Below is a list of the recommended hardware for the OpenShift install. Consider these minimum values - the more the better. Disks must be [SSD/NVME](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/etcd/index#recommended-etcd-practices_etcd-practices).

| Host                      | Count | CPU | Memory | Install Disk | Secondary Disk  | 
| ---                       | ---   | --- | ---    | ---          | ---             |
| installation              | 1     | 4   | 16 GB  | 60 GB        | ---             |
| hub                       | 1     | 16  | 64 GB  | 120 GB       | 1 TB            |
| spoke-control-plane       | 3     | 16  | 32 GB  | 120 GB       | ---             |
| spoke-worker              | 3     | 16  | 64 GB  | 120 GB       | 1 TB (opt)      |

The installation host and hub host can either be a virtual machine or a bare metal host. 

1 x 10 GbE is required for the hosts. To perform more advanced networking, 4 x 10 GbE is recommended for bond0/bond1

This hardware spec also assumes your storage is handled elsewhere and is already in place. If ODF will be used for storage on the spoke cluster, then worker machines would need additional CPU, RAM and an extra SSD/NVME drive in the worker machines to form a storage cluster.

!!! note
    Prefer to have the same vendor and generation CPU architectures across all machines. Worker node sizing dictates workload max sizing.

There is an ability to run the control plane nodes as schedulable, making them effectively worker nodes as well. This is discouraged unless required by the environment constraints, such as testing to run edge systems.

!!! tip "etcd Disk Requirements"
    Run etcd on a block device that can write at least 50 IOPS of 8KB sequentially, including fdatasync, in under 10ms.

## Required Storage

If you are not using ODF, an external storage provider is required.

* Storage provider must provide an OpenShift-compatible Container Storage Interface (CSI) driver
* RO, RWX block/file storage required
* Object storage required

!!! tip "OpenShift Version Capatibility"
    Validate with your storage provider that their CSI is compatible with the stable version of OpenShift. Some vendors CSI driver lag support. 

## Required Network

For a typical POC environment, a `/27` network will suffice for all the required hosts as there are 30 usable addresses. This is assuming an installation host, a hub cluster for ACM and one spoke 3/3 cluster. You will need IPs for: 

* Installation Host
* Hub Cluster 
* Spoke Cluster API VIP
* Spoke Cluster Ingress VIP
* Spoke Control Plane x 3/6
* Spoke Worker x 3/6

> They should all be on the same network and/or vlan.

## Gather the Information

Below are the values an enterprise typically has to gather or create for installing OpenShift.

| Field                             | Example Value                 | Description                                          |
| ---                               | ---                           | ---                                                  |
| **Base Domain**                   | cluster.example.com           | Name of the domain                                   |
| **Machine Subnet**                | 10.1.0.0/24 (vlan - 123)      | Subnet and vlan for all machines/VIPs in cluster     |
| **DNS**                           | dns1.example.com,etc          | IP or hostname for the DNS hosts                     |
| **NTP**                           | ntp1.example.com,etc          | IP or hostname for the NTP hosts                     |
| <h3>Hub Cluster</h3>              |                               |                                                      |
| **Hub Cluster Name**              | hub                           | Name of the cluster                                  |
| **Hub Cluster Suffix**            | hub.cluster.example.com       | cluster name + base domain, for easier notation      |
| **Hub Pod Subnet**                | 10.128.0.0/14                 | Subnet for pod SDN                                   |
| **Pod Subnet - Host Prefix**      | 23                            | Host prefix for Subnet for pod SDN                   |
| **Hub Service Subnet**            | 172.30.0.0/16                 | Subnet for service SDN                               |
| **Hub IP**                        | 10.1.0.3                      | VIP for the MetalLB API Endpoint *                   |
| <h3>Spoke Cluster</h3>            |                               |                                                      |
| **Spoke Cluster Name**            | spoke                         | Name of the cluster                                  |
| **Spoke Cluster Suffix**          | spoke.cluster.example.com     | cluster name + base domain, for easier notation      |
| **Spoke Pod Subnet**              | 10.128.0.0/14                 | Subnet for pod SDN                                   |
| **Pod Subnet - Host Prefix**      | 23                            | Host prefix for Subnet for pod SDN                   |
| **Spoke Service Subnet**          | 172.30.0.0/16                 | Subnet for service SDN                               |
| **Spoke API VIP**                 | 10.1.0.4                      | VIP for the MetalLB API Endpoint *                   |
| **Spoke Ingress VIP**             | 10.1.0.5                      | VIP for the MetalLB Ingress Endpoint *               |

### SDN Subnet Overlaps

The Pod Subnet and Service Subnet are run in the software defined network (SDN) of the cluster. If those values conflict with existing subnets **and** your pods in the cluster will want to route to those outside services with conflicting IPs, you will need to provide different subnets. It's much easier to ensure they are unique.

### Pod Subnet and Host Prefix Explanation

If we are just doing a small cluster, think 500 pods per node and 6 nodes is 3000 IPs. You could use a `/20` and then each node would have a `/23`. `10.128.0.0/20` with `hostPrefix: 23`.

```
10.128.0.0/23    (10.128.0.0 – 10.128.1.255)
10.128.2.0/23    (10.128.2.0 – 10.128.3.255)
10.128.4.0/23    (10.128.4.0 – 10.128.5.255)
10.128.6.0/23    (10.128.6.0 – 10.128.7.255)
10.128.8.0/23    (10.128.8.0 – 10.128.9.255)
10.128.10.0/23   (10.128.10.0 – 10.128.11.255)
10.128.12.0/23   (10.128.12.0 – 10.128.13.255)
10.128.14.0/23   (10.128.14.0 – 10.128.15.255)
```

> [Pod Subnet - Host Prefix](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/installing_on_bare_metal/index#CO63-10)
> [CIDR Subnet Reference](https://docs.netgate.com/pfsense/en/latest/network/cidr.html#understanding-cidr-subnet-mask-notation)

### Machine Information Required

Typically, machines will have more than one NIC and these will be setup in a bond. Please collect the interface names and MAC addresses for ALL NICS and the install disk location on the machines. You provide the hostnames, IPs. IPs need to be located in the machine configuration subnet used on the install. Below is an example table of values needed for collection. 

| Hostname      | Interface | MAC Address        | Bond  | Bond IP    | Disk Hint | BMC IP     |
| ---           | ---       | ---                | ---   | ---        | ---       | ---        |
| installation  | eno1      | A0-B1-C2-D3-E4-E0 | ---   | 10.1.0.2   | /dev/sda  | ---        |
| hub           | eno1      | A0-B1-C2-D3-E4-E1 | ---   | 10.1.0.3   | /dev/sda  | 10.2.0.3   |
| spoke-cp01    | eno1      | A0-B1-C2-D3-E4-E1 | bond0 | 10.1.0.11  | /dev/sda  | 10.2.0.11  |
|               | eno2      | A0-B1-C2-D3-E4-E2 | bond0 |            |           |            |
|               | eno3      | A0-B1-C2-D3-E4-E3 | bond1 |            |           |            |
|               | eno4      | A0-B1-C2-D3-E4-E4 | bond1 |            |           |            |
| spoke-cp02    | eno1      | A0-B1-C2-D3-E4-E5 | bond0 | 10.1.0.12  | /dev/sda  | 10.2.0.12  |
|               | eno2      | A0-B1-C2-D3-E4-E6 | bond0 |            |           |            |
|               | eno3      | A0-B1-C2-D3-E4-E7 | bond1 |            |           |            |
|               | eno4      | A0-B1-C2-D3-E4-E8 | bond1 |            |           |            |
| spoke-cp03    | eno1      | A0-B1-C2-D3-E4-E9 | bond0 | 10.1.0.13  | /dev/sda  | 10.2.0.13  |
|               | eno2      | A0-B1-C2-D3-E4-EA | bond0 |            |           |            |
|               | eno3      | A0-B1-C2-D3-E4-EB | bond1 |            |           |            |
|               | eno4      | A0-B1-C2-D3-E4-EC | bond1 |            |           |            |
| spoke-w01     | eno1      | A0-B1-C2-D3-E4-F1 | bond0 | 10.1.0.21  | /dev/sda  | 10.2.0.21  |
|               | eno2      | A0-B1-C2-D3-E4-F2 | bond0 |            |           |            |
|               | eno3      | A0-B1-C2-D3-E4-F3 | bond1 |            |           |            |
|               | eno4      | A0-B1-C2-D3-E4-F4 | bond1 |            |           |            |
| spoke-w02     | eno1      | A0-B1-C2-D3-E4-F5 | bond0 | 10.1.0.22  | /dev/sda  | 10.2.0.22  |
|               | eno2      | A0-B1-C2-D3-E4-F6 | bond0 |            |           |            |
|               | eno3      | A0-B1-C2-D3-E4-F7 | bond1 |            |           |            |
|               | eno4      | A0-B1-C2-D3-E4-F8 | bond1 |            |           |            |
| spoke-w03     | eno1      | A0-B1-C2-D3-E4-F9 | bond0 | 10.1.0.23  | /dev/sda  | 10.2.0.23  |
|               | eno2      | A0-B1-C2-D3-E4-FA | bond0 |            |           |            |
|               | eno3      | A0-B1-C2-D3-E4-FB | bond1 |            |           |            |
|               | eno4      | A0-B1-C2-D3-E4-FC | bond1 |            |           |            |

The IPs in the example table represent a bonded IP or if the cluster does not use bonding, that IP that connects the machine to the machine network. Notice all machine IPs are inside of the machine subnet of 10.1.0.0/24 defined above. 

On modern RHEL (RHEL CoreOS included), the names of your NICs aren't the old eth0, eth1 style anymore. They use predictable network interface names, which are generated at boot based on hardware topology and firmware information. Here's the gist of how RHEL decides what your NICs will be called:

* eno1, eno2 → onboard NICs (from BIOS/firmware)
* ens1f0, ens1f1 → PCI Express slots ("s" = slot, "f" = function)
* enp3s0 → PCI bus location (p3 = bus 3, s0 = slot 0)
* enx → if nothing else matches, fall back to the MAC address

So the name is tied to where the NIC is physically, not just "first one detected." 

!!! warning
    If you don't know what the values will be (disk name, NIC names), boot the boxes with a [RHEL ISO](https://access.redhat.com/downloads/content/rhel) and find out. Don't guess.

!!! note
    If you are using a hostname scheme that uses an integer at the end of the name, you should start with 1, not zero. But if you start with zero and you want it to match up with your ending IP, be consistent.

### Network Validation Checklist

| Item                                                        |
| ---                                                         |
| Firewall configured per Red Hat documentation               |
| VLAN IDs for bond0/bond1 trunk ports confirmed on switch    |
| NTP reachability confirmed from all nodes                   |
| MTU confirmed consistent end-to-end                         |
| External load balancer configured (optional)                |


