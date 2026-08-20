# Module 08 — Untrusted Workload — Kernel Isolation

## Brief Overview

Participants deploy a runc pod and a kata pod using the same container image, then run a three-way kernel version comparison: host kernel, runc pod kernel, and kata pod kernel. The runc pod matches the host kernel because it shares the host kernel. The kata pod returns a different — typically older and hardened — guest kernel version, proving that a separate kernel is running inside the kata VM. The module reinforces the isolation boundary by comparing /proc visibility between runc and kata contexts, then closes with a CISO-level blast radius question about guest kernel CVEs.

## Audience and Time

- **Target personas:** Field Engineers, Solution Architects, Consultants
- **Prerequisites for this module:** Module 7 complete; kata SCC granted; kata RuntimeClass present and kata NodePool Ready
- **Estimated duration:** 10 minutes

## Learning Objectives

- Deploy runc and kata pods from the same base image and compare their runtime environments
- Demonstrate kernel-level isolation by comparing host, runc pod, and kata pod kernel versions
- Analyze /proc visibility differences between runc and kata pod contexts
- Analyze the blast radius of a guest kernel CVE under the kata isolation model

## Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Deploy runc and Kata Pods and Run Kernel Version Comparison | 6 min |
| 2 | Compare /proc Visibility and Discuss Blast Radius | 4 min |

## Detailed Steps

### Section 1 — Deploy Pods and Compare Kernel Versions

1. Create a test namespace for this module:
   ```
   oc new-project kata-isolation-test
   ```
2. Grant the kata SCC to the default service account:
   ```
   oc adm policy add-scc-to-user kata -z default -n kata-isolation-test
   ```
3. Apply the runc pod manifest (no runtimeClassName specified — uses the cluster default):
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: runc-pod
     namespace: kata-isolation-test
   spec:
     nodeSelector:
       node-role.kubernetes.io/worker: ""
     containers:
       - name: test
         image: registry.access.redhat.com/ubi9/ubi-minimal:latest
         command: ["sleep", "infinity"]
   ```
4. Apply the kata pod manifest (same base image, kata RuntimeClass specified):
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: kata-pod
     namespace: kata-isolation-test
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
5. Wait for both pods to reach `Running` status:
   ```
   oc get pods -n kata-isolation-test -w
   ```
6. Record the host kernel version by running a debug shell on a worker node:
   ```
   oc debug node/<WORKER_NODE_NAME> -- chroot /host uname -r
   ```
7. Record the runc pod kernel version:
   ```
   oc exec runc-pod -n kata-isolation-test -- uname -r
   ```
8. Record the kata pod kernel version:
   ```
   oc exec kata-pod -n kata-isolation-test -- uname -r
   ```
9. Compare the three values. Confirm that the runc pod version matches the host kernel version — they share the same kernel. Confirm that the kata pod version differs — it is running a separate guest kernel inside the kata VM.
10. Note the significance: the same container image produces two fundamentally different kernel contexts depending on the RuntimeClass used.

### Section 2 — /proc Visibility and Blast Radius

11. Check process visibility from inside the runc pod:
    ```
    oc exec runc-pod -n kata-isolation-test -- ls /proc
    ```
    Observe that /proc is populated with process entries reflecting the shared host kernel namespace.
12. Check process visibility from inside the kata pod:
    ```
    oc exec kata-pod -n kata-isolation-test -- ls /proc
    ```
    Observe that the process list is limited to what is running inside the kata guest VM — the host's processes are not visible.
13. Raise the CISO question for the class: if the kata guest kernel has a CVE that allows a container escape, what is the blast radius?
    - The attacker escapes from the container to the kata guest VM kernel.
    - The kata guest VM is isolated from the host kernel by the hypervisor boundary.
    - The attacker is now inside a guest VM, not on the worker node host.
    - Contrast this with a runc escape: the attacker escapes directly to the host kernel with access to all processes running on the node.
14. Summarize: the kata isolation model does not eliminate CVEs in the guest kernel, but it changes the blast radius — a guest kernel escape reaches a VM boundary, not the host.

## Key Takeaways

- runc pods and the host share the same kernel — `uname -r` inside a runc pod returns the host kernel version.
- kata pods run a separate guest kernel inside a VM — `uname -r` inside a kata pod returns a different version, proving a separate kernel is running.
- /proc visibility inside a kata pod is scoped to the guest VM, not the host — host processes are not visible.
- The kata isolation model changes the blast radius of a container escape: an attacker who escapes the container reaches the guest VM boundary, not the host kernel.
- The guest kernel version difference is verifiable from the terminal — this output is the evidence participants carry into customer conversations.

## Infrastructure Notes

- Both pods use the same container image (`ubi9/ubi-minimal`) to prove that the kernel difference is a function of RuntimeClass, not the image.
- The runc pod should be scheduled on a standard worker node (not the kata NodePool) to produce a valid host kernel comparison. Use `nodeSelector` to ensure this.
- The kata pod requires the `kata=true:NoSchedule` toleration to schedule on the kata NodePool.
- The guest kernel version used by kata is determined by the kata-containers version installed by the operator; it will differ from the RHCOS host kernel.
