# Module 10 — Build Pipeline — Host Escape Attempt

## Brief Overview

Participants deploy a simulated build pod under the kata runtime and execute two deliberate host-escape attempts: a sysrq-trigger write that would reboot a runc host node, and a kernel module load targeting the host filesystem. Both attempts are absorbed by the kata guest VM — neither reaches the host kernel. After each attempt, participants verify that all cluster nodes remain Ready and that neighboring workloads are unaffected. The module reframes the field conversation: the question is not whether malicious build code gets submitted to a pipeline, but what the blast radius is when it does.

## Audience and Time

- **Target personas:** Field Engineers, Solution Architects, Consultants
- **Prerequisites for this module:** Modules 8 and 9 complete; kata RuntimeClass present; kata NodePool Ready; kata SCC configured
- **Estimated duration:** 10 minutes

## Learning Objectives

- Deploy a simulated build pod under the kata runtime to represent an untrusted pipeline workload
- Execute a sysrq-trigger escape attempt and verify that it is absorbed by the kata guest VM
- Execute a kernel module load escape attempt and verify that it is absorbed by the kata guest VM
- Verify that all cluster nodes remain Ready and neighboring workloads are unaffected after both escape attempts
- Analyze the blast radius framing for kata containers in CI/CD pipeline security conversations

## Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Deploy Build Pod and Execute sysrq Escape Attempt | 4 min |
| 2 | Execute Kernel Module Load Attempt and Verify Cluster Health | 4 min |
| 3 | Blast Radius Discussion | 2 min |

## Detailed Steps

### Section 1 — Deploy Build Pod and sysrq Escape Attempt

1. Create a namespace for the escape attempt tests:
   ```
   oc new-project kata-escape-test
   ```
2. Grant the kata SCC to the default service account:
   ```
   oc adm policy add-scc-to-user kata -z default -n kata-escape-test
   ```
3. Deploy a simulated build pod under the kata runtime:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: build-pod
     namespace: kata-escape-test
   spec:
     runtimeClassName: kata
     tolerations:
       - key: kata
         operator: Equal
         value: "true"
         effect: NoSchedule
     containers:
       - name: build
         image: registry.access.redhat.com/ubi9/ubi-minimal:latest
         command: ["sleep", "infinity"]
         securityContext:
           privileged: true
   ```
4. Wait for the build pod to reach `Running` status:
   ```
   oc get pod build-pod -n kata-escape-test -w
   ```
5. Record the current node status before the first escape attempt:
   ```
   oc get nodes
   ```
   Note that all nodes are `Ready`.
6. Execute the sysrq-trigger escape attempt from inside the build pod:
   ```
   oc exec build-pod -n kata-escape-test -- sh -c "echo b > /proc/sysrq-trigger"
   ```
   On a runc pod, writing `b` to sysrq-trigger would immediately reboot the host node. On a kata pod, it reboots the kata guest VM kernel.
7. Immediately check node status:
   ```
   oc get nodes
   ```
   Confirm all nodes remain `Ready` — the sysrq write was absorbed by the kata guest VM and did not affect the host.
8. Redeploy the build pod (the kata guest VM rebooted but the pod will restart):
   ```
   oc delete pod build-pod -n kata-escape-test
   oc apply -f build-pod.yaml
   ```
   Wait for `Running` status before proceeding.

### Section 2 — Kernel Module Load Attempt and Cluster Verification

9. Execute the kernel module load escape attempt from inside the build pod:
   ```
   oc exec build-pod -n kata-escape-test -- sh -c "insmod /host/path/to/module.ko 2>&1"
   ```
   Or attempt to access the host filesystem path where kernel modules reside:
   ```
   oc exec build-pod -n kata-escape-test -- sh -c "ls /proc/sys/kernel/modules_disabled && modprobe nbd 2>&1"
   ```
   Observe the error — the attempt targets the guest VM's kernel, not the host kernel. The host filesystem is not accessible from inside the kata guest.
10. Confirm that the host node's kernel module list is unchanged:
    ```
    oc debug node/<KATA_NODE_NAME> -- chroot /host lsmod | grep nbd
    ```
    The module should not appear — the load attempt did not reach the host kernel.
11. Verify that all cluster nodes remain `Ready`:
    ```
    oc get nodes
    ```
12. Verify that workloads in adjacent namespaces are unaffected:
    ```
    oc get pods -n kata-isolation-test
    oc get pods -n kata-multitenant-test
    ```
    Confirm that pods from Modules 8 and 9 are still `Running`.

### Section 3 — Blast Radius Discussion

13. Summarize the two escape attempts for the class:
    - The sysrq write: under runc, this reboots the host node and takes down all workloads on it; under kata, it reboots only the kata guest VM.
    - The kernel module load: under runc, a privileged pod with host filesystem access can load arbitrary kernel modules; under kata, the attempt targets the guest VM kernel and cannot reach the host.
14. Reframe the field conversation: the question is not whether malicious build code gets submitted to a pipeline (it will). The question is what the blast radius is when it does. Under kata, the blast radius is the guest VM; under runc, it is the host node and all co-located workloads.
15. Note for the class: this makes kata containers a natural fit for CI/CD build pipelines where untrusted or third-party code executes — a common pattern in FSI, SaaS platforms, and multi-tenant developer environments.

## Key Takeaways

- A sysrq-trigger reboot attempt from a kata pod reboots the kata guest VM, not the host node — all cluster nodes remain Ready.
- A kernel module load attempt from a kata pod targets the guest VM kernel — the host kernel and host filesystem are not accessible.
- Both escape attempts are absorbed by the hypervisor boundary between the kata guest VM and the host node.
- The blast radius framing is more useful in customer conversations than a claim that kata prevents all escapes: kata changes what an attacker reaches when an escape succeeds, not just whether one is possible.
- Kata containers are a natural fit for CI/CD build pipelines where untrusted code executes in a shared cluster environment.

## Infrastructure Notes

- The build pod requires `privileged: true` to simulate a worst-case scenario where the build system grants elevated permissions; this is intentional for the escape demonstration.
- The sysrq-trigger attempt will cause the kata guest VM to reboot, which may cause the pod to restart; this is expected behavior, not a cluster problem.
- The kernel module load command may vary by the specific modules available in the UBI image; the lab guide author should test the exact command before the lab and use a module that is clearly absent from the host.
- After the escape attempts, the kata NodePool nodes must be verified `Ready` before Module 11 begins.
