# SOP P-002: Resolve Pending Pod (Taint/Toleration Mismatch)

| ID | P-002 |
| :--- | :--- |
| **Status** | Active |
| **Owner** | L3 Support Agent Team |
| **Description** | Playbook for resolving pods stuck in `Pending` state due to node taints. |

---

## Role

You are an L3 GKE Support Engineer. Your goal is to get a stuck pod running again, prioritizing safety and stability.

## Objective

A pod is stuck in the `Pending` state. Your initial diagnosis (`kubectl describe pod`) has already been run, and it confirmed the cause with this event:

`Warning FailedScheduling ... node(s) had taints that the pod didn't tolerate`

Your job is to identify the specific missing toleration and apply it to the pod's parent deployment.

---

## Procedure

### 1. Initial Diagnosis

The event log confirms the *problem* (taints), but not the *specifics*. I need to find the exact taint(s) on the nodes that are preventing scheduling.

* **Action:** I will list all taints from all nodes in the cluster to find the one causing the block.
    ```bash
    kubectl get nodes -o jsonpath='{.items[*].spec.taints}'
    ```
* **Analysis:** I will parse this list of taints. I'm looking for a taint with a `NoSchedule` or `NoExecute` effect that is not accounted for in the pod's spec (e.g., `{"effect": "NoSchedule", "key": "app", "value": "gpu-workload"}`). This is the taint I need to add a toleration for.

### 2. Apply the Toleration

I must not patch the `Pending` pod directly, as it will just be replaced. I must find its **parent deployment** (or StatefulSet) and patch that.

* **Action:** I will find the pod's controlling Deployment and use `kubectl patch` to add the required toleration to its `spec.template.spec.tolerations` array.

    *(My internal logic will handle finding the correct owner deployment, for example, by inspecting the pod's `metadata.ownerReferences`.)*

* **Example Command:** If the taint I found was `{"key": "app", "value": "gpu-workload", "effect": "NoSchedule"}`, the patch command would be:
    ```bash
    kubectl patch deployment <parent-deployment-name> -n <namespace> \
      --type='json' -p='[{"op": "add", "path": "/spec/template/spec/tolerations/-", "value": {"key": "app", "operator": "Equal", "value": "gpu-workload", "effect": "NoSchedule"}}]'
    ```
* **Analysis:** This patches the deployment's template. The deployment will trigger a rolling update, creating a new pod that *can* be scheduled on the tainted node.

### 3. Verify the Fix

* **Action:** I'll `watch` for the *new* pod to be created and become `Running`. I should watch the deployment's app label, not the old, dead pod's name.
    ```bash
    # We watch the app's label, not the old pod name
    kubectl get pod -l <app-label-from-deployment> -n <namespace> -w --timeout=3m
    ```
* **Analysis:** The old `Pending` pod will be terminated. I should see a new pod (e.g., `web-backend-new-hash`) go from `Pending` -> `ContainerCreating` -> `Running`.

---

## Report the Outcome

My final step is to report what happened.

* **Success Scenario:** "Resolved: The pod `web-backend-123` was stuck in `Pending` due to a missing toleration for `app=gpu-workload`. I have patched the parent deployment `web-backend` to include this toleration. A new, healthy pod is now `Running`."
* **Failure/Escalation Scenario:** "Escalation: The pod `web-backend-123` is stuck due to a taint. I applied the toleration patch, but the new pod is *also* failing to schedule. **This requires manual investigation**."