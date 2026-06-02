# Advanced Cluster Management (ACM)

[Red Hat ACM Documentation](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/latest/html-single/install)

Red Hat Advanced Cluster Management (ACM) provides multicluster lifecycle management, governance, and observability across OpenShift and Kubernetes clusters. Installing ACM also automatically installs the multicluster engine operator.

!!! note
    Only one ACM hub cluster can exist per OpenShift cluster.

## Prerequisites

- A supported version of OpenShift Container Platform
- Cluster administrator privileges
- Storage configured on the cluster

## Install the Operator

Create the namespace, operator group, and subscription:

```shell
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: open-cluster-management
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: open-cluster-management
  namespace: open-cluster-management
spec:
  targetNamespaces:
    - open-cluster-management
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: acm-operator-subscription
  namespace: open-cluster-management
spec:
  sourceNamespace: openshift-marketplace
  source: redhat-operators
  channel: release-2.12
  installPlanApproval: Automatic
  name: advanced-cluster-management
EOF
```

Wait for the operator to install:

```shell
oc get csv -n open-cluster-management -w
```

The `PHASE` should show `Succeeded`.

## Create the MultiClusterHub

Once the operator is ready, create the `MultiClusterHub` resource to deploy the ACM hub:

```shell
cat <<EOF | oc apply -f -
apiVersion: operator.open-cluster-management.io/v1
kind: MultiClusterHub
metadata:
  name: multiclusterhub
  namespace: open-cluster-management
spec: {}
EOF
```

!!! note
    It can take up to 10 minutes for the hub to finish deploying all components.

Monitor the status:

```shell
oc get mch -n open-cluster-management -w
```

The status should eventually show `Running`.

## Verify the Installation

```shell
oc get pods -n open-cluster-management
```

Access the ACM console through the OpenShift console. A new menu entry for **All Clusters** appears in the top navigation bar.

You can also get the ACM route directly:

```shell
oc get route multicloud-console -n open-cluster-management -o jsonpath='{.spec.host}'
```

## Infrastructure Node Placement

If you have infrastructure nodes and want ACM components to run on them instead of worker nodes, add the following to your subscription before applying:

```yaml
spec:
  config:
    nodeSelector:
      node-role.kubernetes.io/infra: ""
    tolerations:
      - key: node-role.kubernetes.io/infra
        effect: NoSchedule
        operator: Exists
```

And add the node selector to the `MultiClusterHub`:

```yaml
spec:
  nodeSelector:
    node-role.kubernetes.io/infra: ""
```

## Import a Managed Cluster

After the hub is running, you can import existing clusters. From the ACM console, navigate to **Infrastructure** > **Clusters** > **Import cluster** and follow the prompts.

To import via CLI, create a `ManagedCluster` and `KlusterletAddonConfig`:

```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: my-cluster
spec:
  hubAcceptsClient: true
```

```shell
oc apply -f managedcluster.yaml
```

Then retrieve the import command to run on the target cluster:

```shell
oc get secret my-cluster-import -n my-cluster -o jsonpath='{.data.import\.yaml}' | base64 -d | oc apply -f - --kubeconfig=<managed-cluster-kubeconfig>
```
