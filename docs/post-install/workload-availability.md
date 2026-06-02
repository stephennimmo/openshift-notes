# Workload Availability

[Workload Availability for Red Hat OpenShift Documentation](https://docs.redhat.com/en/documentation/workload_availability_for_red_hat_openshift/latest/html/remediation_fencing_and_maintenance/)

Workload Availability involves a set of operators that work together to detect unhealthy nodes, remediate them automatically, and rebalance pod placement across the cluster.

| Operator                       | Purpose                                                                              |
| ---                            | ---                                                                                  |
| Node Health Check Operator     | Monitors node conditions and triggers remediation when a node becomes unhealthy      |
| Self Node Remediation Operator | Reboots unhealthy nodes automatically (installed as the default remediation provider) |
| Kube Descheduler Operator      | Evicts pods so the scheduler can rebalance them across nodes                         |

## Self Node Remediation Operator

[Documentation](https://docs.redhat.com/en/documentation/workload_availability_for_red_hat_openshift/latest/html/remediation_fencing_and_maintenance/self-node-remediation-operator-remediate-nodes)

The Self Node Remediation Operator runs on every node as a DaemonSet and reboots nodes that are identified as unhealthy. It minimizes downtime for stateful applications and ReadWriteOnce (RWO) volumes. It works regardless of the management interface (IPMI, BMC) or cluster installation type.

The operator determines its remediation strategy based on available watchdog devices:

1. Hardware watchdog device (most reliable)
2. Software watchdog (`softdog`)
3. Software reboot (fallback)

### Install

The Self Node Remediation Operator is automatically installed when you install the Node Health Check Operator. If you need to install it independently:

```shell
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-workload-availability
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: workload-availability-operator-group
  namespace: openshift-workload-availability
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: self-node-remediation-operator
  namespace: openshift-workload-availability
spec:
  channel: stable
  installPlanApproval: Automatic
  name: self-node-remediation-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### Verify

```shell
oc get csv -n openshift-workload-availability
oc get daemonset -n openshift-workload-availability
```

The DaemonSet should have pods running on every node in the cluster.

Confirm the auto-created remediation template and config:

```shell
oc get selfnoderemediationtemplate -n openshift-workload-availability
oc get selfnoderemediationconfig -n openshift-workload-availability
```

## Node Health Check Operator

[Documentation](https://docs.redhat.com/en/documentation/workload_availability_for_red_hat_openshift/latest/html/remediation_fencing_and_maintenance/node-health-check-operator)

The Node Health Check Operator monitors node conditions and creates remediation requests when nodes become unhealthy. It automatically installs the Self Node Remediation Operator as the default remediation provider.

!!! warning
    The Node Health Check Operator cannot be used on Red Hat OpenShift Service on AWS (ROSA) clusters due to preinstalled machine health checks.

### Install

```shell
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-workload-availability
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: workload-availability-operator-group
  namespace: openshift-workload-availability
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: node-health-check-operator
  namespace: openshift-workload-availability
spec:
  channel: stable
  installPlanApproval: Automatic
  name: node-health-check-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### Verify

```shell
oc get csv -n openshift-workload-availability
oc get deployment -n openshift-workload-availability
```

### Configure a NodeHealthCheck

A default `NodeHealthCheck` CR is created automatically targeting worker nodes. You can customize it or create additional ones.

!!! note
    Do not select both worker and control-plane nodes in the same `NodeHealthCheck` CR. Create separate CRs for each group.

```yaml
apiVersion: remediation.medik8s.io/v1alpha1
kind: NodeHealthCheck
metadata:
  name: nhc-control-plane
spec:
  selector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
  minHealthy: 51%
  unhealthyConditions:
    - type: Ready
      status: "False"
      duration: 300s
    - type: Ready
      status: Unknown
      duration: 300s
  remediationTemplate:
    apiVersion: self-node-remediation.medik8s.io/v1alpha1
    kind: SelfNodeRemediationTemplate
    name: self-node-remediation-automatic-strategy-template
    namespace: openshift-workload-availability
```

```yaml
apiVersion: remediation.medik8s.io/v1alpha1
kind: NodeHealthCheck
metadata:
  name: nhc-worker
spec:
  selector:
    matchExpressions:
      - key: node-role.kubernetes.io/worker
        operator: Exists
  minHealthy: 51%
  unhealthyConditions:
    - type: Ready
      status: "False"
      duration: 300s
    - type: Ready
      status: Unknown
      duration: 300s
  remediationTemplate:
    apiVersion: self-node-remediation.medik8s.io/v1alpha1
    kind: SelfNodeRemediationTemplate
    name: self-node-remediation-automatic-strategy-template
    namespace: openshift-workload-availability
```

Key fields:

- **minHealthy** - Minimum percentage (or count) of healthy nodes required before remediation starts. Default is 51%.
- **unhealthyConditions** - Conditions and durations that mark a node as unhealthy. Default is `Ready=False` or `Ready=Unknown` for 300 seconds.
- **remediationTemplate** - Which remediation provider to use.
- **pauseRequests** - Array of strings to pause new remediations while allowing ongoing ones to complete.

### Escalating Remediations

You can chain multiple remediation strategies with timeouts. If the first strategy does not restore the node, the next one is tried:

```yaml
apiVersion: remediation.medik8s.io/v1alpha1
kind: NodeHealthCheck
metadata:
  name: nhc-worker-escalating
spec:
  selector:
    matchExpressions:
      - key: node-role.kubernetes.io/worker
        operator: Exists
  minHealthy: 51%
  unhealthyConditions:
    - type: Ready
      status: "False"
      duration: 300s
    - type: Ready
      status: Unknown
      duration: 300s
  escalatingRemediations:
    - remediationTemplate:
        apiVersion: self-node-remediation.medik8s.io/v1alpha1
        kind: SelfNodeRemediationTemplate
        name: self-node-remediation-automatic-strategy-template
        namespace: openshift-workload-availability
      order: 0
      timeout: 300s
    - remediationTemplate:
        apiVersion: self-node-remediation.medik8s.io/v1alpha1
        kind: SelfNodeRemediationTemplate
        name: self-node-remediation-resource-deletion-template
        namespace: openshift-workload-availability
      order: 1
      timeout: 300s
```

## Kube Descheduler Operator

[Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/{{ ocp_version }}/html/nodes/controlling-pod-placement-onto-nodes-scheduling#nodes-descheduler)

The Descheduler runs periodically and evicts pods that violate scheduling rules so the default scheduler can reschedule them onto more appropriate nodes. This is useful after node maintenance, cluster scaling, or changes to affinity/taint rules.

!!! note
    The descheduler does not schedule pods — it only evicts. The default scheduler then places them on better-fit nodes.

### Install

```shell
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-kube-descheduler-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-kube-descheduler-operator
  namespace: openshift-kube-descheduler-operator
spec:
  targetNamespaces:
    - openshift-kube-descheduler-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-kube-descheduler-operator
  namespace: openshift-kube-descheduler-operator
spec:
  channel: stable
  installPlanApproval: Automatic
  name: cluster-kube-descheduler-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### Verify

```shell
oc get csv -n openshift-kube-descheduler-operator
```

### Configure the Descheduler

Create a `KubeDescheduler` CR. Only one is allowed per cluster and it must be named `cluster`:

```yaml
apiVersion: operator.openshift.io/v1
kind: KubeDescheduler
metadata:
  name: cluster
  namespace: openshift-kube-descheduler-operator
spec:
  deschedulingIntervalSeconds: 3600
  profiles:
    - AffinityAndTaints
    - LifecycleAndUtilization
  mode: Predictive
```

```shell
oc apply -f kubedescheduler.yaml
```

### Descheduler Profiles

| Profile                      | Description                                                                                     |
| ---                          | ---                                                                                             |
| `AffinityAndTaints`         | Evicts pods that violate inter-pod anti-affinity, node affinity, and node taint rules           |
| `TopologyAndDuplicates`     | Evicts pods to balance topology domain constraints and spread duplicates across nodes            |
| `SoftTopologyAndDuplicates` | Similar to above but for soft (preferred) topology constraints. Mutually exclusive with `TopologyAndDuplicates` |
| `LifecycleAndUtilization`   | Evicts pods from overutilized nodes and long-running pods to rebalance resource usage           |
| `DevPreviewLongLifecycle`   | Targets long-running pods for eviction. Mutually exclusive with `LifecycleAndUtilization`       |
| `EvictPodsWithLocalStorage` | Allows eviction of pods that use local storage (emptyDir volumes)                               |
| `EvictPodsWithPVC`          | Allows eviction of pods that use PersistentVolumeClaims                                         |

### Descheduler Modes

- **Predictive** - The descheduler simulates evictions and reports which pods would be evicted without actually evicting them. Useful for dry-run validation.
- **Automatic** - The descheduler actively evicts pods based on the configured profiles.

### OpenShift Virtualization

[Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/{{ ocp_version }}/html/virtualization/virtual-machines#virt-enabling-descheduler-evictions)

If you are running OpenShift Virtualization, use the `DevPreviewLongLifecycle` profile. This profile handles long-running VM workloads and triggers live migrations instead of pod deletions when rebalancing.

```yaml
apiVersion: operator.openshift.io/v1
kind: KubeDescheduler
metadata:
  name: cluster
  namespace: openshift-kube-descheduler-operator
spec:
  deschedulingIntervalSeconds: 3600
  profiles:
    - DevPreviewLongLifecycle
  mode: Automatic
```

Each `VirtualMachine` must be annotated to opt in to descheduler evictions. Add the annotation to the VM template spec:

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
