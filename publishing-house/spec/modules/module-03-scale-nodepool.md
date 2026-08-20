# Module 03 — Scale a NodePool

## Brief Overview

Participants scale the worker NodePool and observe how ARO HCP translates a single declarative Kubernetes API call into an Azure VM creation — without the participant ever touching the Azure compute API directly. After watching the new node appear in both `oc get nodes` and `az vm list`, participants scale back to the baseline replica count. The round-trip reinforces that the NodePool is the customer's operational surface and that ARO HCP handles all Azure-layer consequences of NodePool changes.

## Audience and Time

- **Target personas:** Field Engineers, Solution Architects, Consultants
- **Prerequisites for this module:** Module 2 complete; default worker NodePool identified; `oc` and Azure CLI authenticated
- **Estimated duration:** 10 minutes

## Learning Objectives

- Scale a NodePool replica count using the `oc` CLI
- Observe how ARO HCP translates a NodePool change into an Azure VM creation
- Verify the new worker node in both the Kubernetes API and the Azure resource group
- Demonstrate NodePool as the customer's complete operational surface for worker capacity management

## Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Scale the NodePool Up and Observe Azure Consequence | 7 min |
| 2 | Scale Back to Baseline and Confirm | 3 min |

## Detailed Steps

### Section 1 — Scale the NodePool Up

1. Record the current worker node count before scaling:
   ```
   oc get nodes -l node-role.kubernetes.io/worker
   ```
   Note the number of `Ready` worker nodes.
2. Record the current VM list in the resource group:
   ```
   az vm list --resource-group <LAB_RESOURCE_GROUP> --output table --query "[].{Name:name, Size:hardwareProfile.vmSize}"
   ```
3. Scale the NodePool replica count up by one:
   ```
   oc scale nodepool <NODEPOOL_NAME> -n <HOSTED_CLUSTER_NAMESPACE> --replicas=<CURRENT_COUNT+1>
   ```
4. Immediately watch the NodePool status update:
   ```
   oc get nodepool <NODEPOOL_NAME> -n <HOSTED_CLUSTER_NAMESPACE> -w
   ```
   Observe the `replicas` field change and the `status.conditions` progression as ARO HCP begins provisioning.
5. In a second terminal, watch for the new node to appear:
   ```
   oc get nodes -w
   ```
6. Cross-reference the Azure CLI to see the new VM being created:
   ```
   az vm list --resource-group <LAB_RESOURCE_GROUP> --output table --query "[].{Name:name, ProvisioningState:provisioningState}"
   ```
   Observe the new VM in `Creating` or `Succeeded` state — confirming that the NodePool API call produced an Azure compute consequence.
7. Wait for the new node to reach `Ready` status in `oc get nodes`. Note the time from `oc scale` command to `Ready` node — typical provisioning time for discussion.
8. Confirm the new VM appears in the Azure resource group:
   ```
   az vm list --resource-group <LAB_RESOURCE_GROUP> --output table
   ```
   The VM count should now be one higher than recorded in step 2.

### Section 2 — Scale Back to Baseline

9. Scale the NodePool back to the original replica count:
   ```
   oc scale nodepool <NODEPOOL_NAME> -n <HOSTED_CLUSTER_NAMESPACE> --replicas=<ORIGINAL_COUNT>
   ```
10. Watch the node drain and disappear:
    ```
    oc get nodes -w
    ```
11. Confirm the VM count in Azure returns to baseline:
    ```
    az vm list --resource-group <LAB_RESOURCE_GROUP> --output table
    ```
12. Confirm all remaining nodes are `Ready`:
    ```
    oc get nodes
    ```
13. Note for the class: at no point did the participant issue an `az vm create` or `az vm delete` command. The NodePool is the complete operational surface — ARO HCP handles the Azure API layer.

## Key Takeaways

- A single `oc scale nodepool` command is the complete operational action; ARO HCP handles all Azure API calls required to fulfill it.
- The Azure CLI confirms that NodePool replica changes have real Azure compute consequences — VMs appear and disappear in the resource group.
- The NodePool is the customer's operational surface for worker capacity; no direct Azure compute API access is needed or expected.
- The round-trip (scale up, observe, scale down) establishes the pattern that will be reused in Module 4 when creating a dedicated kata NodePool.

## Infrastructure Notes

- Scaling up by one node typically takes 5–10 minutes on Azure; if the lab schedule is tight, the instructor may initiate the scale command and observe partial output rather than waiting for full `Ready` status.
- Participants should not scale to zero — this would remove all worker capacity needed for subsequent modules.
- The `<CURRENT_COUNT+1>` value must be computed by the participant from the `oc get nodepool` or `oc get nodes` output rather than being pre-specified.
- Ensure the Azure identity used by participants has permission to observe VM creation via `az vm list` — read access is sufficient.
