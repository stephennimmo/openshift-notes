# Hub Agent Based Install

[Red Hat Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer)

A Single Node OpenShift (SNO) cluster runs the control plane and workloads on a single host. It is suited for edge deployments, labs, and resource-constrained environments.

|         | Minimum |
| ---     | ---     |
| vCPU    | 8       |
| Memory  | 16 GB   |
| Storage | 120 GB  |

## Prerequisites

Complete the [prerequisites](prerequisites.md) and set up the [installation host](installation-host.md).

## DNS

All DNS records point to the single node's IP address. No load balancer is required.

| Record                                    | Value          |
| ---                                       | ---            |
| `api.<cluster_name>.<base_domain>`        | `<node_ip>`   |
| `api-int.<cluster_name>.<base_domain>`    | `<node_ip>`   |
| `*.apps.<cluster_name>.<base_domain>`     | `<node_ip>`   |
| `<hostname>.<cluster_name>.<base_domain>` | `<node_ip>`   |

A reverse PTR record for the node IP is also required.

## Create the Configurations

### Create the Working Directory

```shell
mkdir -p sno && cd sno
git init
echo "install/" > .gitignore
touch install-config.yaml
touch agent-config.yaml
git add -A
git commit -m "repo initialized"
```

### Create the Install Config

Key differences from a multi-node install:

- `controlPlane.replicas` must be `1`
- `compute[0].replicas` must be `0`
- `platform` must be `none: {}`
- No `apiVIP` or `ingressVIP` fields

```yaml
apiVersion: v1
baseDomain: ocp.basedomain.com
metadata:
  name: sno
controlPlane:
  name: master
  architecture: amd64
  hyperthreading: Enabled
  replicas: 1
compute:
  - name: worker
    architecture: amd64
    hyperthreading: Enabled
    replicas: 0
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  machineNetwork:
    - cidr: 10.1.0.0/24
  networkType: OVNKubernetes
  serviceNetwork:
    - 172.30.0.0/16
platform:
  none: {}
pullSecret: 'value from ~/.pull_secret'
sshKey: 'value from ~/.ssh/ocp_ed25519.pub'
```

> If your environment requires a proxy, append the same `proxy` and `additionalTrustBundle` sections as described in the [multi-node install](cluster-install.md).

### Create the Agent Config

The agent-config has a single host entry. The `rendezvousIP` is the node's IP address.

#### Static IP

For static networking, include the full `networkConfig`:

```yaml
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: sno
rendezvousIP: 10.1.0.50
additionalNtpSources:
  - 0.us.pool.ntp.org
  - 1.us.pool.ntp.org
hosts:
  - hostname: sno
    role: master
    rootDeviceHints:
      deviceName: "/dev/sda"
    interfaces:
      - name: eno1
        macAddress: A1:B2:3C:4D:5E:01
    networkConfig:
      interfaces:
        - name: eno1
          type: ethernet
          state: up
          mac-address: A1:B2:3C:4D:5E:01
          ipv4:
            enabled: true
            address:
              - ip: 10.1.0.50
                prefix-length: 24
            dhcp: false
          ipv6:
            enabled: false
      dns-resolver:
        config:
          server:
            - dns1.basedomain.com
            - dns2.basedomain.com
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.1.0.1
            next-hop-interface: eno1
            table-id: 254
```

## Generate the ISO

```shell
#!/bin/bash
rm -rf install
mkdir install
cp install-config.yaml agent-config.yaml install
openshift-install agent create image --dir=install
```

## Boot and Install

Boot the host from the generated ISO. Since there is only one node, it serves as both the rendezvous host and the bootstrap host.

Monitor the install:

```shell
openshift-install agent wait-for bootstrap-complete --dir=install
openshift-install agent wait-for install-complete --dir=install
```

The kubeadmin credentials and kubeconfig are written to the `install` directory on completion.

## Validate the Install

Login to the Cluster:

```shell
oc login --server=https://api.cluster.basedomain.com:6443 -u kubeadmin -p <password>
```

### Test Connectivity

```shell
oc debug node/<worker-node-name> -- chroot /host \
  podman pull registry.redhat.io/ubi9/ubi:latest
```

Cleanup the leftover install and configuration pods:

```shell
oc delete pods --all-namespaces --field-selector=status.phase=Succeeded
oc delete pods --all-namespaces --field-selector=status.phase=Failed
```

For troubleshooting install issues, see [Install Troubleshooting](install-troubleshooting.md).
