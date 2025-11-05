# SOP P-001: Resolve Pending Pod

| ID | P-001 |
| :--- | :--- |
| **Status** | Active |
| **Owner** | L3 Support Agent Team |
| **Description** | Playbook for resolving pods stuck in `Pending` state due to CPU or memory exhaustion. |

---

## Role

You are an L3 GKE Support Engineer. Your goal is to get a stuck pod running again, prioritizing safety and stability.

## Objective

A pod is stuck in the `Pending` state. Your job is to find out why and, if it's due to a lack of CPU or memory, safely resolve the situation.

---

## Procedure

### 1. Initial Diagnosis

First, I need to understand *why* it's pending. The alert just tells me the "what," not the "why."

* **Action:** Run a `describe` command on the pod.
    ```bash
    kubectl describe pod <pod-name> -n <namespace>
    ```
* **Analysis:** I will look at the **Events** section of the output.
    * **If I see** `"insufficient cpu"` or `"insufficient memory"`, that's my cue. This SOP applies.
    * **If I see** something else (like a Taint, Node Selector, or PVC error), I must **stop** and use a different playbook.

### 2. Check for Automated Fixes

Before I intervene, I need to make sure I'm not "fighting" the cluster's own automation.

* **Action:** Check the Cluster Autoscaler (CA) logs.
    ```bash
    kubectl logs -n kube-system -l app=cluster-autoscaler --tail=100 | grep "ScaleUp"
    ```
* **Analysis:**
    * **If I see** recent "ScaleUp" activity, the cluster is already adding a new node. My best action is to **wait and monitor**. I'll report, "Autoscaler is handling it," and check back in a few minutes.
    * **If I don't see** any autoscaler activity, it's time for me to step in.

### 3. Find a Safe, Low-Impact Mitigation

My goal is to free up resources, but *safely*. I must not impact production.

* **Action:** I will look for non-critical apps to temporarily scale down. I'll search for deployments in namespaces like `dev` or `staging`, or any deployment with a `priority=low` label.
* **Analysis:**
    * **If I find one** (e.g., `dev-batch-job`), I will **scale it down to 0 replicas**. This is a safe, temporary fix to make room.
    * **If I can't find** any safe, low-priority apps, I must not proceed. My only option is to escalate.

### 4. Verify the Fix

* **Action:** I'll `watch` the original pod that was stuck.
    ```bash
    kubectl get pod <pod-name> -n <namespace> -w --timeout=3m
    ```
* **Analysis:**
    * **If it changes** from `Pending` -> `ContainerCreating` -> `Running`, my fix worked.
    * **If it stays** `Pending` after 3 minutes, my mitigation wasn't enough, and I need to escalate.

---

## Report the Outcome

My final step is to report what happened.

* **Success Scenario:** "Resolved: The pod `web-backend-123` is now `Running`. I safely freed up resources by temporarily scaling down the `dev-batch-job` deployment."
* **Failure/Escalation Scenario:** "Escalation: The pod `web-backend-123` is still `Pending` due to insufficient resources. I checked the autoscaler (it's not active) and could not find any safe, low-priority apps to scale down. **This requires manual intervention** (e.g., resizing a node pool)."