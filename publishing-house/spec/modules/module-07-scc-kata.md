# Module 07 — SCC + Kata — Trigger, Diagnose, Fix

## Brief Overview

Participants deliberately deploy a kata pod without SCC configuration and observe the resulting Forbidden admission error — the most common Sandboxed Containers failure reported in the field. The module then walks through diagnosing the error using pod events and `oc adm policy who-can`, and fixes it by granting the kata SCC to the service account. The module closes with a production-readiness question about whether granting the SCC to the default service account is appropriate versus using dedicated least-privilege accounts.

## Audience and Time

- **Target personas:** Field Engineers, Solution Architects, Consultants
- **Prerequisites for this module:** Module 6 complete; kata RuntimeClass present; kata NodePool nodes in `Ready` state
- **Estimated duration:** 10 minutes

## Learning Objectives

- Deploy a kata pod without SCC configuration and observe the Forbidden admission error
- Diagnose the SCC admission failure using pod events and `oc adm policy who-can`
- Configure the kata SCC grant to resolve the admission error and deploy the pod successfully
- Analyze the security trade-off between granting SCC to the default service account versus a dedicated least-privilege account

## Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Trigger the SCC Admission Failure | 4 min |
| 2 | Diagnose and Fix the SCC Configuration | 6 min |

## Detailed Steps

### Section 1 — Trigger the SCC Admission Failure

1. Create a test namespace for this module:
   ```
   oc new-project kata-scc-test
   ```
2. Apply a kata pod manifest that specifies the kata RuntimeClass but does not include any SCC configuration:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: kata-test-pod
     namespace: kata-scc-test
   spec:
     runtimeClassName: kata
     tolerations:
       - key: kata
         operator: Equal
         value: "true"
         effect: NoSchedule
     containers:
       - name: test
         image: registry.access.redhat.com/ubi9/ubi-minimal:latest
         command: ["sleep", "infinity"]
   ```
3. Apply the manifest:
   ```
   oc apply -f kata-test-pod.yaml
   ```
4. Check the pod status:
   ```
   oc get pod kata-test-pod -n kata-scc-test
   ```
   Observe that the pod is not in `Running` state.
5. Examine the pod events:
   ```
   oc describe pod kata-test-pod -n kata-scc-test
   ```
   Locate the `Warning` event containing a `Forbidden` message referencing an SCC — this is the admission failure.
6. Note the exact SCC name referenced in the Forbidden message — this is the SCC that must be granted.

### Section 2 — Diagnose and Fix

7. Use `oc adm policy who-can` to understand the current SCC grants:
   ```
   oc adm policy who-can use scc kata -n kata-scc-test
   ```
   Confirm that the `default` service account in `kata-scc-test` is not listed.
8. Check what SCC is required by the kata RuntimeClass:
   ```
   oc get scc kata -o yaml
   ```
   Review the SCC definition — note which capabilities or sysctls it grants that ordinary SCCs do not.
9. Grant the kata SCC to the default service account in the test namespace:
   ```
   oc adm policy add-scc-to-user kata -z default -n kata-scc-test
   ```
10. Delete and re-apply the test pod to trigger a fresh admission evaluation:
    ```
    oc delete pod kata-test-pod -n kata-scc-test
    oc apply -f kata-test-pod.yaml
    ```
11. Watch the pod come up:
    ```
    oc get pod kata-test-pod -n kata-scc-test -w
    ```
    Confirm the pod reaches `Running` status.
12. Verify that the pod is running under the kata runtime:
    ```
    oc exec kata-test-pod -n kata-scc-test -- uname -r
    ```
    The kernel version returned should differ from the host kernel — confirming that the kata guest VM is running.
13. Raise the production consideration: granting the SCC to the `default` service account means any pod in the namespace using the default service account can request the kata SCC. Is this appropriate? Discuss the alternative: create a dedicated service account for kata workloads and grant the SCC only to that account.

## Key Takeaways

- Deploying a kata pod without the kata SCC grant produces a `Forbidden` admission error — this is the most common Sandboxed Containers failure encountered in the field.
- Pod events (`oc describe pod`) and `oc adm policy who-can` are the correct diagnostic tools for SCC admission failures.
- The fix is an explicit SCC grant to the service account running the kata workload.
- Granting the kata SCC to the `default` service account is the fastest fix but grants the SCC to all pods in the namespace; production deployments should use dedicated service accounts with least-privilege SCC grants.
- The SCC admission check happens before the container runtime is invoked — the pod never reaches the kata runtime if the SCC check fails.

## Infrastructure Notes

- The `kata` SCC is created by the Sandboxed Containers operator when the CSV reaches `Succeeded` in Module 5. Confirm it exists before starting this module:
  ```
  oc get scc kata
  ```
- The test namespace `kata-scc-test` should be cleaned up after the module or left for reference by subsequent modules; confirm with the lab guide author whether to include a cleanup step.
- Participants need `cluster-admin` or equivalent privileges to run `oc adm policy add-scc-to-user`.
