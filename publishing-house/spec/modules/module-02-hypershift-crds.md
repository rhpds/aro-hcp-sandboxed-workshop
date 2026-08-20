# Module 02 — Explore HyperShift CRDs — Where Is the Control Plane?

## Brief Overview

Participants move from the Azure CLI evidence of Module 1 into the Kubernetes API layer, inspecting the HyperShift CRDs that model the ARO HCP cluster. The HostedCluster object confirms that a control plane exists but runs in Red Hat's infrastructure — the namespaces that would host it in a self-managed cluster are empty. The NodePool object defines the operational boundary: everything above the NodePool is Red Hat's responsibility; everything below is the customer's. The module closes by confirming that kubeconfig and Azure credential secrets are short-lived and operator-managed, not static.

## Audience and Time

- **Target personas:** Field Engineers, Solution Architects, Consultants
- **Prerequisites for this module:** Module 1 complete; `oc` CLI authenticated to the ARO HCP cluster
- **Estimated duration:** 10 minutes

## Learning Objectives

- Explore the HostedCluster CRD to understand how ARO HCP models a managed control plane
- Verify that the control plane namespaces within the customer cluster are empty
- Analyze the NodePool object to identify the customer's operational boundary
- Verify that kubeconfig and Azure credential secrets are short-lived and operator-managed

## Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Inspect the HostedCluster CRD and Control Plane Namespaces | 5 min |
| 2 | Examine the NodePool Object and Credential Lifecycle | 5 min |

## Detailed Steps

### Section 1 — HostedCluster CRD and Control Plane Namespaces

1. List the custom resource definitions installed by HyperShift:
   ```
   oc get crd | grep hypershift
   ```
   Identify `hostedclusters.hypershift.openshift.io` and `nodepools.hypershift.openshift.io`.
2. Describe the HostedCluster resource for the lab cluster:
   ```
   oc get hostedcluster -A
   oc describe hostedcluster <CLUSTER_NAME> -n <HOSTED_CLUSTER_NAMESPACE>
   ```
3. Note the `status.conditions` section — confirm the cluster reports `Available` and observe the `controlPlaneEndpoint` field, which points to an API server endpoint not backed by VMs in the customer resource group.
4. List the namespaces that would host control plane components in a self-managed cluster:
   ```
   oc get ns | grep -E 'openshift-etcd|openshift-kube-apiserver|openshift-kube-controller-manager|openshift-kube-scheduler'
   ```
   Confirm these namespaces are either absent or empty of running pods.
5. Explicitly check for etcd pods:
   ```
   oc get pods -n openshift-etcd 2>&1
   ```
   Expect "No resources found" or a namespace-not-found error — etcd runs in Red Hat's infrastructure, not in the customer cluster.

### Section 2 — NodePool Object and Credential Lifecycle

6. List the NodePool objects:
   ```
   oc get nodepool -A
   ```
7. Describe the default worker NodePool:
   ```
   oc describe nodepool <NODEPOOL_NAME> -n <HOSTED_CLUSTER_NAMESPACE>
   ```
8. Identify in the NodePool spec: the `replicas` field (customer-controlled), the `platform.aws` or `platform.azure` VM type field (customer-controlled), and the `clusterName` reference (links back to the HostedCluster).
9. Draw the operational boundary for the class: the NodePool and everything beneath it (worker VMs, node configuration, workloads) is the customer's operational surface. The HostedCluster and everything above it (API server, etcd, controller manager, scheduler) is Red Hat's responsibility.
10. Inspect the kubeconfig secret managed by the operator:
    ```
    oc get secret -n <HOSTED_CLUSTER_NAMESPACE> | grep kubeconfig
    oc describe secret <KUBECONFIG_SECRET_NAME> -n <HOSTED_CLUSTER_NAMESPACE>
    ```
    Observe that the secret has a short expiry annotation or rotation timestamp — it is operator-managed, not static.
11. Inspect the Azure credential secret similarly:
    ```
    oc get secret -n <HOSTED_CLUSTER_NAMESPACE> | grep azure
    ```
    Note that credentials are injected and rotated by the operator rather than being long-lived static values.
12. Summarize for the class: the HyperShift CRDs provide a clean Kubernetes-native API surface for a managed cluster. Customers interact with NodePool; Red Hat manages everything above it.

## Key Takeaways

- The HostedCluster CRD provides a Kubernetes-native representation of the ARO HCP managed control plane; the control plane itself runs in Red Hat's infrastructure, not in customer namespaces.
- Control plane namespaces (etcd, kube-apiserver, kube-controller-manager, kube-scheduler) are empty or absent in the customer cluster — confirming the Azure CLI evidence from Module 1.
- The NodePool is the customer's operational boundary: replicas, VM type, and node configuration are customer-controlled; everything above is Red Hat-managed.
- Kubeconfig and Azure credential secrets are short-lived and operator-managed, aligning with the Workload Identity model observed in Module 1.

## Infrastructure Notes

- The `<HOSTED_CLUSTER_NAMESPACE>` value depends on the ARO HCP provisioning setup; it may be `clusters` or a named namespace — confirm before the lab and provide to participants.
- The HyperShift CRDs must be present in the cluster; they are installed as part of ARO HCP provisioning.
- If credential secrets have already been rotated before the lab session, the `describe` output will still show the `managedFields` and last-updated timestamp that demonstrates operator management.
