# Module 04 — Create a Kata-Dedicated NodePool

## Brief Overview

Participants create a second NodePool targeting the Standard_D4s_v3 VM SKU, which is the minimum SKU that supports nested virtualization on Azure — a requirement for kata containers. Two manifest details receive close attention: the node label that will bind KataConfig in Module 6, and the NoSchedule taint that prevents ordinary runc workloads from being scheduled onto kata nodes. The module closes by verifying that KVM is available on the new nodes, confirming the hypervisor support needed by the kata runtime.

## Audience and Time

- **Target personas:** Field Engineers, Solution Architects, Consultants
- **Prerequisites for this module:** Modules 1–3 complete; default worker NodePool at baseline; `oc` and Azure CLI authenticated
- **Estimated duration:** 10 minutes

## Learning Objectives

- Create a second NodePool targeting a VM SKU that supports nested virtualization
- Configure a node label on the NodePool that will bind KataConfig in a later module
- Configure a NoSchedule taint to prevent runc workloads from landing on kata nodes
- Verify that KVM is available on the new nodes

## Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Create the Kata-Dedicated NodePool Manifest and Apply | 5 min |
| 2 | Verify New Nodes and Confirm KVM Availability | 5 min |

## Detailed Steps

### Section 1 — Create and Apply the NodePool Manifest

1. Review the VM SKU requirement: Standard_D4s_v3 provides 4 vCPUs and 16 GiB RAM with nested virtualization enabled on Azure. This is the minimum SKU for kata containers.
2. Create the NodePool manifest. Examine each field carefully before applying:
   ```yaml
   apiVersion: hypershift.openshift.io/v1beta1
   kind: NodePool
   metadata:
     name: kata-nodepool
     namespace: <HOSTED_CLUSTER_NAMESPACE>
   spec:
     clusterName: <CLUSTER_NAME>
     replicas: 2
     platform:
       type: Azure
       azure:
         vmsize: Standard_D4s_v3
     nodeLabels:
       kata.kubernetes.io/nodepool: "true"
     taints:
       - key: kata
         value: "true"
         effect: NoSchedule
   ```
3. Note the `nodeLabels` field: the label `kata.kubernetes.io/nodepool: "true"` must match the `kataConfigPoolSelector` in the KataConfig manifest that will be applied in Module 6. If these do not match exactly, KataConfig installation fails silently.
4. Note the `taints` field: the `kata=true:NoSchedule` taint prevents runc workloads from being scheduled onto kata nodes without an explicit toleration. This protects runc workloads from incurring kata overhead and isolates the node pool for kata-only use.
5. Apply the manifest:
   ```
   oc apply -f kata-nodepool.yaml
   ```
6. Watch the NodePool status:
   ```
   oc get nodepool kata-nodepool -n <HOSTED_CLUSTER_NAMESPACE> -w
   ```
   Observe the replica count and conditions progressing as Azure provisions the new VMs.

### Section 2 — Verify New Nodes and Confirm KVM Availability

7. Wait for the new nodes to appear and reach `Ready` status:
   ```
   oc get nodes -l kata.kubernetes.io/nodepool=true
   ```
   Confirm that the node label is present on the new nodes.
8. Confirm the taint is applied:
   ```
   oc describe node <KATA_NODE_NAME> | grep -A5 Taints
   ```
   Verify `kata=true:NoSchedule` is listed.
9. Open a debug shell on one of the new kata nodes:
   ```
   oc debug node/<KATA_NODE_NAME>
   ```
10. Inside the debug shell, check for KVM device availability:
    ```
    ls -la /dev/kvm
    ```
    A successful result confirms the hypervisor extension is available and nested virtualization is enabled on the Azure VM.
11. Exit the debug shell.
12. Confirm in the Azure CLI that two new VMs of the expected SKU appear in the resource group:
    ```
    az vm list --resource-group <LAB_RESOURCE_GROUP> --output table --query "[].{Name:name, Size:hardwareProfile.vmSize}"
    ```

## Key Takeaways

- Standard_D4s_v3 is the minimum Azure VM SKU for kata containers because it supports nested virtualization (KVM).
- The `kata.kubernetes.io/nodepool: "true"` node label must exactly match the `kataConfigPoolSelector` in the KataConfig manifest — a mismatch causes silent installation failure.
- The `kata=true:NoSchedule` taint prevents runc workloads from landing on kata nodes, protecting them from unintended overhead.
- Verifying `/dev/kvm` availability on the new nodes confirms the hypervisor support required by the kata runtime before attempting installation.
- This NodePool is used by all subsequent modules (5–10) — it must be healthy before proceeding.

## Infrastructure Notes

- Standard_D4s_v3 VMs must be available in the Azure region used by the lab. Confirm regional availability before the lab.
- NodePool provisioning typically takes 5–10 minutes on Azure. If time is tight, the instructor can initiate the NodePool creation and verify KVM availability after Module 5's operator installation begins (the two can proceed in parallel).
- The kata NodePool should have at least 2 replicas to support the multi-tenant isolation tests in Module 9.
- The node label key `kata.kubernetes.io/nodepool` is used consistently through Modules 4, 6, and all subsequent isolation modules — do not alter it.
