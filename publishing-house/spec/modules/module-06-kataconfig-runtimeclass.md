# Module 06 — Apply KataConfig and Verify RuntimeClass

## Brief Overview

Participants apply a KataConfig manifest and watch the kata runtime install itself across the kata-dedicated NodePool. The critical detail is the `kataConfigPoolSelector` — the label selector that must exactly match the node label set on the kata NodePool in Module 4. A mismatch causes KataConfig to either do nothing or attempt installation on the wrong nodes without producing an obvious error. The module observes the full MachineConfigPool update cycle in real time: nodes cordon, drain, reboot, and return Ready. Verification happens at two layers — the kata RuntimeClass appearing in the Kubernetes API, and a node debug shell confirming containerd's runtime registry includes the kata handler.

## Audience and Time

- **Target personas:** Field Engineers, Solution Architects, Consultants
- **Prerequisites for this module:** Module 5 complete; Sandboxed Containers operator CSV in `Succeeded` state; kata NodePool nodes in `Ready` state
- **Estimated duration:** 10 minutes

## Learning Objectives

- Configure KataConfig with a `kataConfigPoolSelector` that binds to the kata-dedicated NodePool
- Observe the MachineConfigPool update cycle as kata binaries are installed on designated nodes
- Verify that the kata RuntimeClass appears in the Kubernetes API after KataConfig installation completes
- Verify that containerd's runtime registry on a kata node includes the kata-runtime handler

## Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Apply KataConfig and Watch MachineConfigPool Update Cycle | 6 min |
| 2 | Verify RuntimeClass and Containerd Runtime Registry | 4 min |

## Detailed Steps

### Section 1 — Apply KataConfig

1. Review the KataConfig manifest before applying. Examine the `kataConfigPoolSelector` field carefully:
   ```yaml
   apiVersion: kataconfiguration.openshift.io/v1
   kind: KataConfig
   metadata:
     name: cluster-kataconfig
   spec:
     kataConfigPoolSelector:
       matchLabels:
         kata.kubernetes.io/nodepool: "true"
   ```
2. Confirm that the label value `kata.kubernetes.io/nodepool: "true"` exactly matches the `nodeLabels` set on the kata NodePool in Module 4. Emphasize to the class: a label key or value mismatch here causes KataConfig to silently skip installation — there is no admission error.
3. Apply the KataConfig:
   ```
   oc apply -f kataconfig.yaml
   ```
4. Watch the KataConfig status update:
   ```
   oc get kataconfig cluster-kataconfig -w
   ```
   Observe the `status.installationStatus` fields as the operator begins configuring nodes.
5. Watch the MachineConfigPool for the kata node pool:
   ```
   oc get mcp -w
   ```
   Observe the `UPDATED`, `UPDATING`, and `DEGRADED` column changes as nodes enter the update cycle.
6. Watch the nodes cordon, drain, reboot, and return to Ready:
   ```
   oc get nodes -l kata.kubernetes.io/nodepool=true -w
   ```
   Observe node status transitions: `Ready` → `SchedulingDisabled` (cordon) → `NotReady` (reboot) → `Ready` (complete).

### Section 2 — Verify RuntimeClass and Containerd Runtime Registry

7. Once all kata nodes return to `Ready`, verify that the kata RuntimeClass has been created:
   ```
   oc get runtimeclass
   ```
   Confirm that a RuntimeClass named `kata` (or `kata-qemu`) appears in the output.
8. Describe the RuntimeClass to inspect its handler name:
   ```
   oc describe runtimeclass kata
   ```
   Note the `handler` field — this is the name containerd uses to look up the kata runtime in its configuration.
9. Open a debug shell on one of the kata nodes:
   ```
   oc debug node/<KATA_NODE_NAME>
   ```
10. Inside the debug shell, inspect the containerd runtime configuration:
    ```
    cat /host/etc/crio/crio.conf.d/50-kata-runtimes.conf
    ```
    Or, depending on the containerd configuration path:
    ```
    cat /host/etc/containerd/config.toml | grep -A5 kata
    ```
    Confirm that the kata handler is registered as a runtime in the containerd configuration.
11. Exit the debug shell.
12. Confirm the total number of installed nodes matches the kata NodePool replica count:
    ```
    oc get kataconfig cluster-kataconfig -o jsonpath='{.status.installationStatus.completed.completedNodesList}'
    ```

## Key Takeaways

- The `kataConfigPoolSelector` in KataConfig must exactly match the node label set on the kata NodePool — a mismatch causes silent installation failure with no admission error.
- The MachineConfigPool update cycle (cordon → drain → reboot → Ready) is the mechanism by which kata binaries are installed on designated nodes; this is expected behavior, not a problem.
- Verification requires checking at two layers: the Kubernetes API (RuntimeClass object) and the node filesystem (containerd runtime configuration).
- Once KataConfig installation is complete, the cluster is ready for kata workloads; the next three modules use this runtime to test isolation properties.

## Infrastructure Notes

- The MachineConfigPool update cycle reboots nodes — this is expected. Worker nodes in the kata NodePool should not be running critical workloads during the lab.
- The full update cycle for a 2-node kata NodePool typically takes 5–8 minutes; if this exceeds the module time, the instructor can initiate the apply and proceed with discussion while the cycle completes.
- The containerd configuration path may vary by OCP version; confirm the correct path before the lab and update the lab guide accordingly.
- The label key `kata.kubernetes.io/nodepool` must be consistent across Module 4 (NodePool nodeLabels) and Module 6 (KataConfig kataConfigPoolSelector).
