# Module 09 — Multi-Tenant Isolation

## Brief Overview

Two simulated tenant kata pods run on the same physical node, each carrying a distinct secret in their environment. Participants attempt cross-tenant inspection from Tenant A — attempting to read Tenant B's processes and environment variables — and find nothing. The same inspection is then repeated using runc pods, where Tenant B's secrets are visible through the shared /proc filesystem. Seeing both outputs back to back makes the difference between soft and hard multi-tenancy immediately concrete and provides a directly reproducible proof for customer conversations.

## Audience and Time

- **Target personas:** Field Engineers, Solution Architects, Consultants
- **Prerequisites for this module:** Module 8 complete; kata RuntimeClass present; kata NodePool Ready; kata SCC configured
- **Estimated duration:** 10 minutes

## Learning Objectives

- Deploy two simulated tenant kata pods on the same physical node, each with distinct secrets
- Demonstrate that cross-tenant process and environment variable inspection fails under kata isolation
- Deploy two runc pods and demonstrate that cross-tenant secrets are visible through the shared /proc filesystem
- Analyze the distinction between soft multi-tenancy (namespace isolation) and hard multi-tenancy (kernel-level isolation) using observed terminal output

## Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Deploy Kata Tenants and Attempt Cross-Tenant Inspection | 5 min |
| 2 | Deploy runc Tenants and Observe Secret Visibility | 5 min |

## Detailed Steps

### Section 1 — Kata Tenant Isolation

1. Create a namespace for the multi-tenant isolation test:
   ```
   oc new-project kata-multitenant-test
   ```
2. Grant the kata SCC to the default service account:
   ```
   oc adm policy add-scc-to-user kata -z default -n kata-multitenant-test
   ```
3. Deploy Tenant A kata pod with a distinct secret in its environment:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: kata-tenant-a
     namespace: kata-multitenant-test
   spec:
     runtimeClassName: kata
     tolerations:
       - key: kata
         operator: Equal
         value: "true"
         effect: NoSchedule
     containers:
       - name: app
         image: registry.access.redhat.com/ubi9/ubi-minimal:latest
         command: ["sleep", "infinity"]
         env:
           - name: TENANT_SECRET
             value: "tenant-a-secret-value"
   ```
4. Deploy Tenant B kata pod with a different secret:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: kata-tenant-b
     namespace: kata-multitenant-test
   spec:
     runtimeClassName: kata
     tolerations:
       - key: kata
         operator: Equal
         value: "true"
         effect: NoSchedule
     containers:
       - name: app
         image: registry.access.redhat.com/ubi9/ubi-minimal:latest
         command: ["sleep", "infinity"]
         env:
           - name: TENANT_SECRET
             value: "tenant-b-secret-value"
   ```
5. Confirm both pods are `Running` and note the node they are scheduled on:
   ```
   oc get pods -n kata-multitenant-test -o wide
   ```
   If possible, confirm both are on the same kata node — if not, proceed regardless; the inspection result is the same.
6. Exec into Tenant A and attempt to list processes:
   ```
   oc exec kata-tenant-a -n kata-multitenant-test -- ps aux
   ```
   Observe that only processes inside Tenant A's guest VM are visible.
7. Attempt to enumerate /proc entries from Tenant A to find Tenant B's process IDs:
   ```
   oc exec kata-tenant-a -n kata-multitenant-test -- ls /proc | grep -E '^[0-9]+'
   ```
8. For any process ID found, attempt to read its environment:
   ```
   oc exec kata-tenant-a -n kata-multitenant-test -- cat /proc/1/environ 2>&1
   ```
   Observe that only Tenant A's own environment is accessible — `TENANT_SECRET=tenant-a-secret-value` — and `tenant-b-secret-value` does not appear.

### Section 2 — runc Tenant Comparison

9. Deploy Tenant A runc pod with a distinct secret:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: runc-tenant-a
     namespace: kata-multitenant-test
   spec:
     nodeSelector:
       node-role.kubernetes.io/worker: ""
     containers:
       - name: app
         image: registry.access.redhat.com/ubi9/ubi-minimal:latest
         command: ["sleep", "infinity"]
         env:
           - name: TENANT_SECRET
             value: "tenant-a-secret-value"
   ```
10. Deploy Tenant B runc pod:
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: runc-tenant-b
      namespace: kata-multitenant-test
    spec:
      nodeSelector:
        node-role.kubernetes.io/worker: ""
      containers:
        - name: app
          image: registry.access.redhat.com/ubi9/ubi-minimal:latest
          command: ["sleep", "infinity"]
          env:
            - name: TENANT_SECRET
              value: "tenant-b-secret-value"
    ```
11. Confirm both runc pods are `Running` on the same worker node:
    ```
    oc get pods -n kata-multitenant-test -o wide
    ```
12. Find Tenant B's PID from Tenant A's /proc:
    ```
    oc exec runc-tenant-a -n kata-multitenant-test -- ls /proc | grep -E '^[0-9]+'
    ```
13. Read Tenant B's environment from Tenant A's context:
    ```
    oc exec runc-tenant-a -n kata-multitenant-test -- cat /proc/<TENANT_B_PID>/environ 2>&1 | tr '\0' '\n'
    ```
    Observe that `tenant-b-secret-value` appears in the output — Tenant A can read Tenant B's environment through the shared /proc filesystem.
14. Compare the two outputs side by side for the class: kata prevents cross-tenant inspection because each kata pod runs in a separate VM with its own kernel; runc allows it because all pods on the node share the same host kernel and /proc namespace.

## Key Takeaways

- Under kata isolation, two pods on the same physical node cannot inspect each other's processes or environment variables — each runs in a separate guest VM with its own kernel namespace.
- Under runc, two pods on the same physical node share the host kernel and /proc — a pod in one namespace can enumerate and read another pod's environment if it can access /proc numerically by PID.
- The difference between soft multi-tenancy (Kubernetes namespace isolation) and hard multi-tenancy (kernel-level isolation) is directly observable from the terminal.
- This module produces a reproducible proof that participants can replay in a customer environment or a recorded demo.
- The kata isolation holds even when two tenants are co-located on the same physical node — placement does not weaken the isolation boundary.

## Infrastructure Notes

- Both kata tenant pods should ideally land on the same kata node to make the strongest co-location argument; use pod affinity rules if needed, but the inspection result is the same regardless.
- Both runc tenant pods should land on the same standard worker node; use pod affinity to ensure this.
- The `tr '\0' '\n'` command in step 13 converts null-byte-separated environment variables to newline-separated output for readability.
- The specific PID for Tenant B (step 13) must be found dynamically by the participant — it cannot be pre-specified in the lab guide.
