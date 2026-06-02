# Networking

## Overview

From a linux configuration perspective, remember that:

- There are **physical connections** (`type: ethernet`)
- Those can be **bonded** (`type: bond`)
    - Bonds have a mode of `802.3ad` (LACP), `balance-xor`, or `active-backup`
- **VLANs** can have a `base-iface` of an existing `ethernet` or `bond`
    - You can have multiple VLANs from a trunked `base-iface`

!!! info "Typical OpenShift Production Setup"
    - **7 NICs**
    - **3 Bonds** — LACP
        - `bond0` — management bond for cluster traffic
        - `bond1` — data bond for pod network
        - `bond2` — storage network
    - **1 BMC interface**

---

## The Full Stack for Underlay Networking

!!! note "Abbreviations"
    - **NNCP** = NodeNetworkConfigurationPolicy
    - **OVN-K** = Open Virtual Networking — Kubernetes (OpenShift's CNI)

| Concept                              | Where      | Managed By |
| ---                                  | ---        | ---        |
| Switch                               | Physical   | Switch     |
| Ethernet                             | Linux      | NNCP       |
| Bond                                 | Linux      | NNCP       |
| OVS Bridge                           | Linux      | NNCP       |
| OVN Bridge Mapping                   | OVN-K      | NNCP       |
| Localnet                             | OVN-K      | NNCP       |
| Cluster User Defined Network (CUDN)  | OVN-K      | —          |
| Network Attachment Definition        | OVN-K      | CUDN       |
| Virtual Ethernet Pair                | Kubernetes | CNI        |

---

## Descriptions

### Switch (Physical Layer 2 device)

**What it is:** The physical top-of-rack or in-rack switch (Arista, MikroTik, Juniper, etc.).

**Role in OpenShift:** Provides the L2 domain where your cluster nodes attach. VLAN tags, trunks, and port-channels live here.

**Implementation:** Actual silicon forwarding Ethernet frames. OVN doesn't know or care about it directly, but VLANs and MTU mismatches will break your cluster if misaligned.

---

### Ethernet (NIC inside the node)

**What it is:** The real network interface card (NIC) in the node. Could be Intel, Mellanox, Solarflare, etc.

**Implementation:** In Linux, shows up as `eno1`, `ens3f0`, etc. Uses a kernel driver (`ixgbe`, `mlx5_core`, `sfc`).

**In OpenShift:** The base network device that NNCP manipulates.

```yaml
- name: <devicename>
  type: ethernet
  state: up
  mac-address: <macaddress>
  ipv4:
    enabled: false
  ipv6:
    enabled: false
```

---

### Bond (Linux bonding or LACP)

**What it is:** A logical link aggregation across multiple physical NICs.

**Implementation:** Linux kernel bonding driver (`bonding.ko`). Supports modes `active-backup`, `802.3ad` (LACP), `balance-xor`.

**In OpenShift:** Configured by the SR-IOV or NMState operators. Exposed in NNCP YAML (`type: bond`, `link-aggregation:`).

```yaml
- name: bond<i>
  type: bond
  state: up
  link-aggregation:
    mode: 802.3ad
    port:
      - <devicename>
      - <devicename2>
    options:
      miimon: "100"
      lacp_rate: fast
  ipv4:
    enabled: false
  ipv6:
    enabled: false
```

---

### VLAN

**What it is:** A logical sub-interface identified by a VLAN ID, created on top of a NIC or bond.

**Role in OpenShift:** Segments traffic into isolated broadcast domains, allowing pods or nodes to connect to different L2 networks over the same physical link.

**Implementation:** Linux kernel `8021q` module creates sub-interfaces (e.g., `bond0.31` for VLAN 31). Configured via NNCP or NetworkManager. VLAN tags are inserted/removed by the kernel before frames leave/arrive on the NIC.

```yaml
- name: bond<i>.<vlanid>
  type: vlan
  state: up
  vlan:
    base-iface: bond<i>
    id: <vlanid>
  ipv4:
    enabled: true
    address:
      - ip: <ipaddress>
        prefix-length: 24
    dhcp: false
  ipv6:
    enabled: false
```

---

### OVS Bridge (Open vSwitch kernel module + daemon)

**What it is:** A software switch running in each node.

**Implementation:** Kernel datapath + `ovs-vswitchd` in userspace. Can do VLAN tagging, port mirroring, QoS.

**In OpenShift:** OVN-Kubernetes uses an OVS bridge (`br-int`) for pod connectivity. If you're doing localnet or external connectivity, OVS bridges handle the attachment to physical NICs or bonds.

```yaml
- name: ovs-bridge-trunk
  type: ovs-bridge
  state: up
  bridge:
    port:
      - name: bond1
    allow-extra-patch-ports: true
    options:
      stp: false
```

---

### OVN Bridge Mapping

**What it is:** A mapping between a logical OVN bridge port and a real OVS bridge/port on the node.

**Implementation:** Managed by the ovn-kubernetes daemonset (`ovn-controller`). Stored in OVSDB.

!!! example
    `bridge-mappings=physnet1:br-ex` tells OVN that traffic for `physnet1` is reachable via the host's `br-ex` bridge.

```yaml
desiredState:
  ovn:
    bridge-mappings:
      - localnet: localnet-bridge-trunk
        bridge: ovs-bridge-trunk
        state: present
```

---

### Localnet (OVN construct)

**What it is:** A special OVN logical switch port type that maps a logical network directly to a physical network.

**Implementation:** In OVN NB DB as `type=localnet`. Bypasses overlay tunnels (Geneve) and uses VLAN-backed segments.

**In OpenShift:** Used for Cluster User Defined Networks (CUDN) with `type: Localnet` so pods connect to a real VLAN instead of overlay.

---

### Cluster User Defined Network (CUDN)

**What it is:** An OpenShift CRD (`ClusterNetwork`) that defines extra networks beyond the default pod network.

**Implementation:** Managed by the Network Operator — writes into OVN NB DB. Can be Geneve overlay or Localnet VLAN-backed.

**In practice:** This is how you say "I want a storage network on VLAN 31, isolated from the default pod net."

```yaml
apiVersion: k8s.ovn.org/v1
kind: ClusterUserDefinedNetwork
metadata:
  name: vlan<vlanid>
spec:
  namespaceSelector:
    matchExpressions:
      - key: kubernetes.io/metadata.name
        operator: In
        values: ["<namespace>", "<namespace2>"]
  network:
    topology: Localnet
    localnet:
      role: Secondary
      physicalNetworkName: localnet-bridge-trunk
      vlan:
        mode: Access
        access:
          id: <vlanid>
      ipam:
        mode: Disabled
```

---

### Network Attachment Definitions (NADs)

**What it is:** A Kubernetes CRD (`network-attachment-definitions.k8s.cni.cncf.io`) provided by Multus.

**Implementation:** NAD describes which CNI plugin (OVN, SR-IOV, macvlan, whereabouts) to call when attaching extra networks to a pod.

**In OpenShift:** Each NAD corresponds to a `net-attach-def` object. Pods reference them via `k8s.v1.cni.cncf.io/networks`.

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vlan<vlanid>
  namespace: <namespace>
  ownerReferences:
    - apiVersion: k8s.ovn.org/v1
      blockOwnerDeletion: true
      controller: true
      kind: ClusterUserDefinedNetwork
      name: vlan<vlanid>
spec:
  config: '
    {
      "cniVersion":"1.0.0",
      "mtu":1500,
      "name":"cluster_udn_vlan<vlanid>",
      "netAttachDefName":"<namespace>/vlan<vlanid>",
      "physicalNetworkName":"localnet-bridge-trunk",
      "role":"secondary",
      "topology":"localnet",
      "type":"ovn-k8s-cni-overlay",
      "vlanID":<vlanid>
    }'
```

---

### Virtual Ethernet Pair (veth)

**What it is:** The plumbing pipe.

**Implementation:** Linux kernel primitive (`ip link add veth0 type veth peer name veth1`).

**In OpenShift:** When a pod is created, Multus/OVN creates a veth pair. One end goes into the pod's netns (`eth0`). The other end plugs into `br-int` (OVS) or another bridge. OVN programs flows to steer traffic between veth, tunnels, and physical NICs.

---

### Pod (Kubernetes abstraction)

**What it is:** A Linux process in a container namespace.

**Networking view:** Sees its veth `eth0`, a private IP, and the routes injected by the CNI plugin.

**In OpenShift:** The pod doesn't know about OVN, Multus, or OVS — it just thinks it has a NIC. Behind the curtain, OVN is doing NAT, overlay encapsulation (Geneve), or VLAN bridging depending on your network config.

---

## NodeNetworkConfigurationPolicy Examples

This is just for setting up your bonds.

??? example "2-eth-bond-lacp-vlan"
    ```yaml
    --8<-- "docs/networking/2-eth-bond-lacp-vlan.yaml"
    ```

??? example "2-eth-bond-active-backup-vlan"
    Another example for setting up your bonds with `active-backup`.

    ```yaml
    --8<-- "docs/networking/2-eth-bond-active-backup-vlan.yaml"
    ```

??? example "ovs-bridge-trunk-nncp"
    This example assumes you have an existing `bond1` on each of your worker nodes. One of the good things about using bonded connections is that if your device names are not consistent across your hardware, you have the ability to obfuscate the differences and name the bonds all the same, thus your additional configurations can be applied to larger groupings of hardware.

    ```yaml
    --8<-- "docs/networking/ovs-bridge-trunk-nncp.yaml"
    ```

---

## ClusterUserDefinedNetwork

Once you have your NNCPs in place and your underlays are created, now you need to take it that last mile to the namespaces.

??? example "cudn-with-ipam"
    This example assumes you have an existing OVN bridge mapping for a localnet connection called `localnet-bridge-trunk`.

    ```yaml
    --8<-- "docs/networking/cudn-with-ipam.yaml"
    ```

??? example "cudn-no-ipam"
    This example assumes you have an existing OVN bridge mapping for a localnet connection called `localnet-bridge-trunk`.

    ```yaml
    --8<-- "docs/networking/cudn-no-ipam.yaml"
    ```
