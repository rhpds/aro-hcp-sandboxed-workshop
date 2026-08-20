# ARO HCP + OpenShift Sandboxed Containers Workshop

<!-- This file is the design document for your lab or demo. -->
<!-- Fill in each section below, or run /rhdp-publishing-house to have the intake skill help. -->
<!-- Sections marked with [brackets] are placeholders — replace with real content. -->
<!-- The validation gate checks for all required sections before submission. -->

## Overview

This 120-minute hands-on lab demonstrates two complementary technologies that address real customer concerns around cloud cost, operational complexity, and workload isolation. Participants first explore ARO HCP's managed control plane model — proving directly from the Azure CLI that master VMs, etcd disks, and control plane load balancers simply do not exist in the customer resource group. They then install OpenShift Sandboxed Containers, configure a kata-dedicated NodePool, and run a structured sequence of isolation tests — kernel version comparisons, cross-tenant inspection attempts, and deliberate host-escape attempts — to produce verifiable proof of the kernel-level isolation boundary.

Participants will: list Azure resources to confirm the absent control plane, inspect HyperShift CRDs to understand the operational boundary, scale a NodePool and observe the Azure consequence, create a kata-dedicated NodePool, install and configure the Sandboxed Containers Operator and KataConfig, reproduce and fix the most common SCC admission failure in the field, compare runc and kata kernel environments, attempt cross-tenant secret extraction, and execute host-escape simulations that are absorbed by the guest VM.

## Target Audience

- **Role:** Field Engineers, Solution Architects, Consultants
- **Experience level:** Intermediate
- **What they already know:** Hands-on OpenShift experience — comfortable with `oc` CLI, namespaces, operators, and YAML manifests
- **What they don't know:** ARO HCP's Hosted Control Plane architecture and operational model; OpenShift Sandboxed Containers / kata containers configuration and isolation properties

## Prerequisites

- None

<!-- Familiarity with OpenShift is assumed as part of the target audience definition, not a formal prerequisite. -->

## Learning Objectives

1. Verify ARO HCP's managed control plane model by inspecting Azure resource groups and HyperShift CRDs
2. Scale a NodePool and correlate the Kubernetes-layer action to its Azure consequence
3. Deploy OpenShift Sandboxed Containers and configure KataConfig on a kata-dedicated NodePool
4. Troubleshoot and resolve common SCC admission failures for kata workloads
5. Demonstrate kernel-level isolation by comparing host, runc, and kata pod kernel environments
6. Analyze the blast radius of multi-tenant workload compromise under runc versus kata isolation

<!-- 6 objectives for 120 minutes — within the guideline of up to 3 per 45 minutes. Each is testable from terminal output. -->

## Content Type

Lab (hands-on)

## Products & Technologies

- ARO HCP (Azure Red Hat OpenShift with Hosted Control Planes)
- OpenShift Sandboxed Containers
- Azure CLI (upstream tooling)
- HyperShift (upstream project — provides HostedCluster and NodePool CRDs)
- kata-containers (upstream runtime — used by OpenShift Sandboxed Containers)

## Module Map

| Module | Title | Duration |
|--------|-------|----------|
| 0 | Introduction and Pre-Flight | 10 min |
| 1 | Inspect the Cluster Resource Group | 10 min |
| 2 | Explore HyperShift CRDs — Where Is the Control Plane? | 10 min |
| 3 | Scale a NodePool | 10 min |
| 4 | Create a Kata-Dedicated NodePool | 10 min |
| 5 | Install the Sandboxed Containers Operator | 10 min |
| 6 | Apply KataConfig and Verify RuntimeClass | 10 min |
| 7 | SCC + Kata — Trigger, Diagnose, Fix | 10 min |
| 8 | Untrusted Workload — Kernel Isolation | 10 min |
| 9 | Multi-Tenant Isolation | 10 min |
| 10 | Build Pipeline — Host Escape Attempt | 10 min |
| 11 | Wrap-Up & Questions | 10 min |
| — | **Total hands-on** | **100 min** |
| — | Intro / pre-flight | ~10 min |
| — | **Wrap-up** | ~10 min |
| — | **Total lab** | **~120 min** |

<!-- Each module 10 minutes. Modules 1–3 cover ARO HCP; Modules 4–10 cover Sandboxed Containers. The two tracks build on each other — the kata NodePool created in Module 4 is used by all subsequent isolation tests. -->

## Difficulty Level

Intermediate

## Environment

**Learner view:** Participants start with a pre-provisioned ARO HCP cluster on Azure. When they log in, the cluster is running and their kubeconfig is pre-configured. The Azure CLI is authenticated with a scoped identity that can read the lab resource group. Worker nodes are running; control plane infrastructure (master VMs, etcd disks, control plane load balancers) is absent from the resource group because it runs in Red Hat's managed infrastructure. There is no pre-installed Sandboxed Containers operator — participants install and configure it as part of the lab.

**Automation needed:** Yes — the ARO HCP cluster must be pre-provisioned before the lab starts, with kubeconfig and Azure CLI credentials delivered to each participant's environment. A standard worker NodePool exists at lab start; participants create the kata-dedicated NodePool themselves in Module 4.

## Infrastructure Requirements

- **Cloud provider:** TBD — confirmed in infrastructure phase
- **Cluster type:** TBD — confirmed in infrastructure phase
- **OCP version:** TBD — confirmed in infrastructure phase
- **Topology:** TBD — confirmed in infrastructure phase
- **Sizing:** TBD — confirmed in infrastructure phase
- **Automation approach:** TBD — confirmed in infrastructure phase
- **AI/MaaS:** TBD — confirmed in infrastructure phase
- **External services:** TBD — confirmed in infrastructure phase
- **AAP version:** TBD — confirmed in infrastructure phase
- **Non-GA products:** TBD — confirmed in infrastructure phase

## Assessment Strategy (Optional)

This is a classic Showroom lab — no solve/validate buttons. Success is trust-based: participants observe expected terminal outputs (absent Azure resources, correct kernel version deltas, blocked cross-tenant inspection, absorbed host-escape attempts) and compare them to the expected results shown in the lab guide.
