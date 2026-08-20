# Module 11 — Wrap-Up and Questions

## Brief Overview

This closing module shifts from doing to synthesizing. Participants have spent 110 minutes at the terminal producing verifiable evidence across two technology tracks. This final ten minutes gives that work a frame they can carry into a customer conversation the next day — connecting each terminal output to a customer concern, identifying where the two technologies reinforce each other, and surfacing the open questions that belong in a production design conversation.

## Audience and Time

- **Target personas:** Field Engineers, Solution Architects, Consultants
- **Prerequisites for this module:** Modules 0–10 complete
- **Estimated duration:** 10 minutes

## Learning Objectives

- Synthesize the ARO HCP findings (Modules 1–3) into a customer-ready summary of the managed control plane value proposition
- Synthesize the Sandboxed Containers findings (Modules 4–10) into a customer-ready summary of the kernel isolation value proposition
- Demonstrate how the two technologies address complementary layers of customer concerns (operational complexity and workload blast radius)
- Analyze which lab outputs are most relevant to FSI, government, and multi-tenant SaaS customer conversations

## Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | ARO HCP Track Synthesis | 3 min |
| 2 | Sandboxed Containers Track Synthesis | 4 min |
| 3 | Open Questions and Field Application | 3 min |

## Detailed Steps

### Section 1 — ARO HCP Track Synthesis

1. Recall the three terminal outputs from Modules 1–3 that serve as direct evidence in customer conversations:
   - `az vm list` showing no master VMs in the resource group (Module 1)
   - `oc describe hostedcluster` showing the control plane endpoint with no backing VMs in the customer resource group (Module 2)
   - `oc scale nodepool` producing an `az vm list` change with no direct Azure API interaction (Module 3)
2. Connect each output to the customer concern it addresses:
   - Absent master VMs: cost reduction at scale, confirmed from Azure CLI
   - Workload Identity bindings: FSI and government credential compliance, no static credentials
   - NodePool as the operational surface: reduced operational scope for the customer team
3. Summarize the ARO HCP value proposition in one sentence the class can use tomorrow: ARO HCP removes the control plane from the customer's operational scope — verifiable from the Azure CLI — while preserving full Kubernetes API access through the NodePool boundary.

### Section 2 — Sandboxed Containers Track Synthesis

4. Recall the four terminal outputs from Modules 7–10 that serve as direct evidence in customer conversations:
   - `oc describe pod` showing the Forbidden SCC error and fix (Module 7) — addresses the "how hard is this to operate?" question
   - `uname -r` comparison showing different kernel versions (Module 8) — addresses "prove the isolation"
   - Cross-tenant /proc inspection comparison showing kata blocks what runc exposes (Module 9) — addresses "prove multi-tenancy"
   - Node status check after escape attempts showing all nodes remain Ready (Module 10) — addresses "what is the blast radius?"
5. Connect each output to the customer concern it addresses:
   - SCC fix: operational familiarity — kata workloads use standard OpenShift admission controls
   - Kernel version delta: hard evidence of a separate guest kernel, not just a marketing claim
   - Cross-tenant /proc comparison: distinguishes soft from hard multi-tenancy with observable output
   - Post-escape node health: blast radius evidence for security architects and CISOs
6. Summarize the Sandboxed Containers value proposition in one sentence: kata containers provide a hypervisor-backed isolation boundary that changes the blast radius of a workload compromise from the host node to a guest VM — verifiable from the terminal.

### Section 3 — Open Questions and Field Application

7. Raise the production questions that were surfaced during the lab and left deliberately open:
   - Should `installPlanApproval` be `Automatic` or `Manual` in production? (Module 5)
   - Should the kata SCC be granted to the `default` service account or a dedicated least-privilege account? (Module 7)
   - What is the guest kernel CVE blast radius, and how does the organization's patching cadence account for the kata guest kernel version? (Module 8)
8. Identify which customer segments are most likely to lead with each value proposition:
   - ARO HCP control plane cost and compliance: FSI, government, large multi-cluster platform teams
   - Sandboxed Containers multi-tenant isolation: SaaS platforms, CI/CD pipeline operators, shared developer environments
9. Note that the two technologies are complementary: ARO HCP reduces operational overhead at the cluster layer; Sandboxed Containers reduces blast radius at the workload layer. Both can be present in the same deployment.
10. Open for participant questions. Encourage participants to identify one specific customer or prospect conversation where they can use one of the terminal outputs from today's lab as direct evidence.

## Key Takeaways

- The lab produced verifiable terminal outputs for each major claim: absent control plane resources, NodePool as operational boundary, kernel version isolation, cross-tenant inspection blocking, and absorbed host-escape attempts.
- ARO HCP and Sandboxed Containers address different layers: ARO HCP at the cluster operations layer, Sandboxed Containers at the workload isolation layer — they reinforce each other in a multi-tenant platform deployment.
- Production deployments raise open questions about installPlanApproval, SCC grant scope, and guest kernel CVE patching cadence — these are the right questions to surface early in a customer engagement.
- Every output produced today is reproducible in a customer environment or a recorded demo — participants have a reusable proof kit, not just a slide.

## Infrastructure Notes

- No new cluster commands are required in this module — it is synthesis only.
- The instructor should ensure that the Modules 8 and 9 test namespaces are available for reference if participants want to re-run a specific comparison during questions.
- If time permits, participants can be invited to run one of the key verification commands (`uname -r` comparison or cross-tenant /proc check) a final time as a closing demonstration.
