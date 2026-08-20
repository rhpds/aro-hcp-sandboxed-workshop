# Module 01 — Inspect the Cluster Resource Group

## Brief Overview

Participants use the Azure CLI to enumerate all resources in the lab resource group and confirm what is absent: there are no master VMs, no etcd disks, and no control plane load balancers. The first half turns the absence of those resources into a concrete cost calculation. The second half examines the Managed Identity and Workload Identity bindings that provide Azure API access to the cluster without static credentials, establishing why ARO HCP satisfies FSI and government security policies by default.

## Audience and Time

- **Target personas:** Field Engineers, Solution Architects, Consultants
- **Prerequisites for this module:** Module 0 pre-flight complete; Azure CLI authenticated
- **Estimated duration:** 10 minutes

## Learning Objectives

- Verify that master VMs, etcd disks, and control plane load balancers are absent from the customer resource group
- Analyze the cost and operational implications of the absent control plane resources across a multi-cluster deployment
- Explore Managed Identity and Workload Identity bindings that replace static credentials in ARO HCP

## Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | List Resource Group Contents and Identify Absent Resources | 5 min |
| 2 | Examine Managed Identity and Workload Identity Bindings | 5 min |

## Detailed Steps

### Section 1 — List Resource Group Contents

1. List all resources in the lab resource group:
   ```
   az resource list --resource-group <LAB_RESOURCE_GROUP> --output table
   ```
2. Scan the output for VMs. Confirm that only worker VMs appear — there are no master VMs (typically `master-0`, `master-1`, `master-2` in a self-managed cluster).
3. Filter specifically for virtual machines to make the finding explicit:
   ```
   az vm list --resource-group <LAB_RESOURCE_GROUP> --output table --query "[].{Name:name, Size:hardwareProfile.vmSize}"
   ```
4. Confirm that no etcd-related managed disks are present:
   ```
   az disk list --resource-group <LAB_RESOURCE_GROUP> --output table
   ```
   Expect either an empty list or disks associated only with worker nodes — no etcd data disks.
5. Confirm that no control plane load balancers are present:
   ```
   az network lb list --resource-group <LAB_RESOURCE_GROUP> --output table
   ```
6. Work through the cost calculation with the class: a standard self-managed OpenShift cluster requires 3 master VMs. With 15 shared clusters on ARO HCP, that eliminates 45 master VMs from customer resource groups. Note this also eliminates the associated etcd disks and control plane load balancers per cluster.
7. Record the names of the worker VMs for reference in Module 3 (NodePool scaling).

### Section 2 — Examine Managed Identity and Workload Identity Bindings

8. List the managed identities associated with the cluster:
   ```
   az identity list --resource-group <LAB_RESOURCE_GROUP> --output table
   ```
9. Inspect one of the managed identity role assignments to understand what Azure API permissions the cluster identity carries:
   ```
   az role assignment list --assignee <MANAGED_IDENTITY_PRINCIPAL_ID> --output table
   ```
10. From the OpenShift side, inspect the credentials request objects that the cluster uses to obtain scoped Azure credentials:
    ```
    oc get credentialsrequest -n openshift-cloud-credential-operator
    ```
11. Observe that credentials are issued per component and scoped — no single static credential with broad permissions is used.
12. Note for the class: this Workload Identity model makes ARO HCP compliant with FSI and government security policies that prohibit long-lived static credentials with broad Azure API access.

## Key Takeaways

- The customer resource group contains only worker infrastructure — master VMs, etcd disks, and control plane load balancers are absent because they run in Red Hat's managed infrastructure.
- Eliminating the control plane from customer resource groups directly reduces VM count, disk count, and load balancer count — with compounding cost savings at scale across multiple clusters.
- ARO HCP uses Managed Identity and Workload Identity bindings instead of static credentials, satisfying FSI and government security requirements by default.
- The absence of resources is itself a compliance artifact, not just a cost saving.

## Infrastructure Notes

- The Azure CLI identity used in the lab must have at least `Reader` role on the lab resource group to execute `az resource list`, `az vm list`, `az disk list`, and `az network lb list`.
- The `<LAB_RESOURCE_GROUP>` value must be provided to participants at lab start or pre-populated in the lab guide.
- The `<MANAGED_IDENTITY_PRINCIPAL_ID>` value should be discoverable from `az identity list` output — participants look it up rather than having it pre-provided.
