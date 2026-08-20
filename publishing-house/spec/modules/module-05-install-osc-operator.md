# Module 05 — Install the Sandboxed Containers Operator

## Brief Overview

Participants apply a single Subscription manifest that creates the Namespace, OperatorGroup, and Subscription objects required to install the OpenShift Sandboxed Containers operator. While waiting for the ClusterServiceVersion to reach `Succeeded`, the module examines the three custom resources the operator manages: KataConfig (node binary installation), RuntimeClass (containerd registration), and PeerPodConfig (peer pod support for environments without nested virtualization). The module closes with a production-readiness question about the `Automatic` installPlanApproval setting.

## Audience and Time

- **Target personas:** Field Engineers, Solution Architects, Consultants
- **Prerequisites for this module:** Module 4 complete; kata NodePool healthy; `oc` CLI authenticated with cluster-admin privileges
- **Estimated duration:** 10 minutes

## Learning Objectives

- Install the OpenShift Sandboxed Containers operator using a Subscription manifest
- Monitor the ClusterServiceVersion until it reaches Succeeded status
- Explore the three custom resources managed by the operator: KataConfig, RuntimeClass, and PeerPodConfig
- Analyze the security implications of Automatic versus Manual installPlanApproval for production deployments

## Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Apply Subscription Manifest and Watch Installation | 5 min |
| 2 | Explore Operator-Managed Resources and Production Considerations | 5 min |

## Detailed Steps

### Section 1 — Apply Subscription Manifest

1. Review the Subscription manifest before applying. The file contains three objects separated by `---`:
   ```yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: openshift-sandboxed-containers-operator
   ---
   apiVersion: operators.coreos.com/v1
   kind: OperatorGroup
   metadata:
     name: sandboxed-containers-operator-group
     namespace: openshift-sandboxed-containers-operator
   spec:
     targetNamespaces:
       - openshift-sandboxed-containers-operator
   ---
   apiVersion: operators.coreos.com/v1alpha1
   kind: Subscription
   metadata:
     name: sandboxed-containers-operator
     namespace: openshift-sandboxed-containers-operator
   spec:
     channel: stable
     installPlanApproval: Automatic
     name: sandboxed-containers-operator
     source: redhat-operators
     sourceNamespace: openshift-marketplace
   ```
2. Note the `installPlanApproval: Automatic` field — hold this observation for the production discussion in Section 2.
3. Apply the manifest:
   ```
   oc apply -f osc-subscription.yaml
   ```
4. Watch the operator installation progress by monitoring the CSV:
   ```
   oc get csv -n openshift-sandboxed-containers-operator -w
   ```
   Wait for the phase column to show `Succeeded`.
5. While waiting, confirm the InstallPlan was created and approved:
   ```
   oc get installplan -n openshift-sandboxed-containers-operator
   ```

### Section 2 — Explore Operator-Managed Resources

6. Once the CSV reaches `Succeeded`, list the CRDs introduced by the operator:
   ```
   oc get crd | grep kata
   ```
   Identify `kataconfigs.kataconfiguration.openshift.io`.
7. Examine what the KataConfig CRD controls:
   ```
   oc explain kataconfig.spec
   ```
   Note the `kataConfigPoolSelector` field — this is the label selector that binds KataConfig to the kata NodePool created in Module 4.
8. Check whether a RuntimeClass named `kata` exists yet:
   ```
   oc get runtimeclass
   ```
   The RuntimeClass will not exist yet — it is created when KataConfig is applied in Module 6.
9. Check for the PeerPodConfig CRD:
   ```
   oc get crd | grep peerpod
   ```
   Explain to the class: PeerPodConfig enables kata containers in environments where the underlying VMs do not support nested virtualization — the kata VM runs as a peer pod on a separate cloud instance rather than nested on the worker node. This is relevant for Azure environments without nested virtualization support.
10. Return to the production consideration: the Subscription uses `installPlanApproval: Automatic`. Ask the class: in a production cluster, should operator updates be applied automatically? Discuss the trade-off between staying current and the risk of an operator update changing behavior during a maintenance freeze.

## Key Takeaways

- The Subscription manifest is the complete installation surface: Namespace, OperatorGroup, and Subscription in one file.
- The operator manages three key custom resources: KataConfig (installs kata binaries on designated nodes), RuntimeClass (registers the kata handler with containerd), and PeerPodConfig (enables kata in non-nested-virtualization environments).
- KataConfig and RuntimeClass do not exist until a KataConfig object is applied in Module 6 — the operator installs only the machinery, not the runtime itself.
- `Automatic` installPlanApproval is convenient for labs but may not be appropriate for production clusters where operator updates must be change-managed.
- The `kataConfigPoolSelector` field in KataConfig must match the node label set on the kata NodePool in Module 4 — this binding is critical and will be configured in Module 6.

## Infrastructure Notes

- The `redhat-operators` CatalogSource must be present in `openshift-marketplace` for the Subscription to resolve — this is standard in connected ARO HCP clusters.
- If the cluster is in a disconnected or air-gapped environment, the CatalogSource must be mirrored; this lab assumes a connected environment.
- CSV installation typically takes 2–4 minutes; participants use the wait time for the exploration steps in Section 2.
- Cluster-admin privileges are required to create the Namespace, OperatorGroup, Subscription, and to approve InstallPlans.
