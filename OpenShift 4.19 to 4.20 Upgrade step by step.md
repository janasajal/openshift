## OpenShift 4.19 to 4.20 Upgrade: Step-by-Step Guide with Internal Workflow

In this lab, an OpenShift cluster was successfully upgraded from version 4.19 to 4.20. Upgrading OpenShift requires more than simply executing an upgrade command; it involves a carefully orchestrated process to ensure cluster availability while updates are applied.

This lab report details the steps taken to handle `AdminAckRequired` blockers, set upgrade channels, trigger the upgrade, and observe the automated node upgrade process.

---

### Pre-Upgrade Checks

Before initiating the upgrade, several critical checks were performed to ensure cluster health:

* Operators were verified to ensure none were in a "Degraded" state.
* Pod Disruption Budgets (PDBs) were reviewed. (Strict PDBs can cause upgrades to hang, while overly generous PDBs can impact application availability during node draining).
* The cluster was checked for failing pods.
* A diagnostic data collection was run using the `oc adm must-gather` command.

---

1. **Check Current Upgrade Status:**
The current upgrade status of the cluster was verified.

```bash
oc adm upgrade

```

The output indicated that the upgrade was blocked because an administrator acknowledgment was required:
`Upgradeable=False`
`Reason: AdminAckRequired`
`Message: The admissionregistration.k8s.io/v1beta1 group version is deprecated in 4.19 and will be removed in 4.20...`

This blocker is placed intentionally to ensure administrators have accounted for deprecated APIs before upgrading. A Red Hat knowledgebase article link is typically provided in the output to guide the unblocking process.


2. **Acknowledge and Approve Deprecations:**
To proceed, the required acknowledgment was provided by patching the `admin-acks` ConfigMap in the `openshift-config` namespace.

```bash
oc -n openshift-config patch cm admin-acks \
  --patch '{"data":{"ack-4.19-admissionregistration-v1beta1-api-removals-in-4.20":"true"}}' \
  --type=merge

```


3. **Set the Upgrade Channel:** Switching to fast-4.20.
The upgrade channel was configured. For this lab, the `fast` channel was used, though `stable` or `EUS` (Extended Update Support) channels are recommended for production environments.

```bash
oc adm upgrade channel fast-4.20

```

The configuration was verified by running `oc adm upgrade` again, which confirmed the targeted version (4.20.18) was now available.


4. **Trigger the Upgrade:**
The upgrade was initiated, targeting version 4.20.18.

```bash
oc adm upgrade --to=4.20.18

```


5. **Monitor the Upgrade Process:**
The upgrade progress was actively monitored. Instead of leaving the upgrade unobserved, the cluster version status was watched in real-time.

```bash
oc get clusterversion -w

```

The status transitioned to `Working towards 4.20.18`, showing the percentage of completed tasks.


6. **Verify Completion:**
Finally, the completion of the upgrade was verified.

```bash
oc get clusterversion

```

The output confirmed that `AVAILABLE` was `True`, `PROGRESSING` was `False`, and the active cluster version was successfully updated to 4.20.18.


---

### What Happened Under the Hood

Once the upgrade command was executed, the OpenShift automation handled the node-by-node update process:

1. **Control Plane Nodes:** The master nodes were upgraded first. To maintain cluster quorum, they were processed one at a time. Each master node was cordoned, drained of pods, updated, rebooted, had its API restored, and was finally uncordoned.
2. **Worker Nodes:** After the control plane was updated, the worker nodes followed the same sequential process (Cordon -> Drain -> Update -> Reboot -> Uncordoned).

This lab demonstrated that the upgrade process is straightforward provided that API deprecations are properly handled, the correct channel is set, and comprehensive pre-checks are not skipped.
