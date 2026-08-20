# Module 00 — Introduction and Pre-Flight

## Brief Overview

This module opens the 120-minute workshop by establishing the problem context and technology frame. The instructor briefly covers what ARO HCP is, what OpenShift Sandboxed Containers is, and how the two technologies connect to address cloud cost, operational complexity, and workload isolation concerns. The second half of the module is a live pre-flight check: participants run a short series of commands to confirm that their environment is correctly provisioned and ready before the hands-on sequence begins.

## Audience and Time

- **Target personas:** Field Engineers, Solution Architects, Consultants
- **Prerequisites for this module:** None — this is the opening module
- **Estimated duration:** 10 minutes

## Learning Objectives

- Describe the relationship between ARO HCP's managed control plane model and OpenShift Sandboxed Containers' workload isolation model
- Verify that the pre-provisioned ARO HCP cluster is accessible and that the `oc` CLI and Azure CLI are authenticated and ready

## Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Instructor Framing — ARO HCP and Sandboxed Containers | 5 min |
| 2 | Live Pre-Flight Check | 5 min |

## Detailed Steps

### Section 1 — Instructor Framing

1. Instructor explains the two-part structure of the lab: Modules 1–3 focus on ARO HCP's managed control plane architecture; Modules 4–10 focus on OpenShift Sandboxed Containers isolation properties.
2. Instructor frames the connecting thread: ARO HCP removes operational overhead at the control plane layer; Sandboxed Containers reduces blast radius at the workload layer. Both address real concerns raised in FSI, government, and multi-tenant customer conversations.
3. Instructor notes that every claim in the lab will be verified directly from the terminal — no slides substituting for evidence.

### Section 2 — Live Pre-Flight Check

4. Open a terminal in the provided lab environment.
5. Verify `oc` CLI access by running:
   ```
   oc whoami
   oc get nodes
   ```
   Confirm that worker nodes appear in `Ready` status and that no master or infra nodes are listed.
6. Verify Azure CLI authentication by running:
   ```
   az account show
   az group list --output table
   ```
   Confirm that the lab resource group is listed and the identity is scoped correctly.
7. Verify that the Sandboxed Containers operator is **not** yet installed:
   ```
   oc get csv -n openshift-sandboxed-containers-operator 2>&1
   ```
   Expect a "not found" or empty result — confirming the operator will be installed by participants in Module 5.
8. Note the names of the default worker NodePool for use in Module 3:
   ```
   oc get nodepool -A
   ```
9. Confirm that all pre-flight checks pass before proceeding to Module 1. If any check fails, raise the issue with the lab proctor.

## Key Takeaways

- ARO HCP offloads control plane operations to Red Hat's managed infrastructure; participants will prove this from the Azure CLI in Module 1.
- OpenShift Sandboxed Containers provides kernel-level workload isolation; participants will verify this through structured tests in Modules 8–10.
- The two technologies are complementary: ARO HCP reduces operational surface; Sandboxed Containers reduces workload blast radius.
- The lab environment is pre-provisioned; the Sandboxed Containers operator is intentionally absent at the start.

## Infrastructure Notes

- The ARO HCP cluster must be running and participant kubeconfigs must be pre-configured before this module begins.
- The Azure CLI must be authenticated with an identity scoped to the lab resource group (read access is sufficient for Modules 1–3).
- Worker nodes must be in `Ready` status; control plane infrastructure (master VMs, etcd disks, control plane load balancers) must not appear in the resource group.
- No Sandboxed Containers operator or KataConfig should be present at lab start.
