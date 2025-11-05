# SOP P-003: Resolve Pending Pod (Node Selector Mismatch)

| ID | P-003 |
| :--- | :--- |
| **Status** | Active |
| **Owner** | L3 Support Agent Team |
| **Description** | Playbook for resolving pods stuck in `Pending` state due to a `nodeSelector` mismatch. |

---

## Role

You are an L3 GKE Support Engineer. Your goal is to get a stuck pod running again, prioritizing safety and stability.

## Objective

A pod is stuck in the `Pending` state. Your initial diagnosis (`kubectl describe pod`) has already been run, and it confirmed the cause with this event:

`Warning FailedScheduling ... node(s) didn't match node selector`

Your job is to identify the incorrect `nodeSelector` (or `nodeAffinity`) and patch the pod's parent deployment to match existing node labels.

---

## Procedure

### 1. Initial Diagnosis

The event log tells me a `nodeSelector` failed, but not *which one*. I need to inspect the pod's own specification.

* **Action:** Get the `nodeSelector` from the pending pod's YAML.
    ```bash
    kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.nodeSelector}'
    ```
* **Analysis:** This will output the failing key-value pair, for example: `{"disktype":"ssd"}`. This is what the pod *wants*.

### 2. Find the Correct Node Label

Now I know what the pod wants. I need to check what the cluster *actually has*. The user likely made a typo or used the wrong label key.

* **Action:** List the labels on the cluster's nodes to find the correct, existing label.
    ```bash
    # This command shows all labels on all nodes
    kubectl get nodes --show-labels
    ```
* **Analysis:** I'll scan the output for a label that is *similar* to the one the pod wants.
    * **Hypothesis:** The pod wants `disktype: ssd`.
    * **Discovery:** I see the nodes actually have the label `cloud.google.com/gke-local-ssd: "true"`.
    * **Conclusion:** The user configured the wrong label. I need to *replace* `disktype: ssd` with `cloud.google.com/gke-local-ssd: "true"`.

### 3. Apply the Fix (Patch Deployment)

As with other scheduling failures, I must patch the pod's **parent deployment** to make the fix permanent.

* **Action:** I will find the pod's controlling Deployment and use `kubectl patch` to replace the incorrect `nodeSelector` with the correct one.
* **Example Command:**
    ```bash
    # This command replaces the entire nodeSelector map
    kubectl patch deployment <parent-deployment-name> -n <namespace> \
      --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/nodeSelector", "value": {"[cloud.google.com/gke-local-ssd](https://cloud.google.com/gke-local-ssd)": "true"}}]'
    ```
* **Analysis:** The deployment will trigger a rolling update. The new pod created by its ReplicaSet will have the correct `nodeSelector` and will be scheduled successfully.

### 4. Verify the Fix

* **Action:** I'll `watch` for the *new* pod to be created and become `Running`, using the deployment's app label.
    ```bash
    # We watch the app's label, not the old pod name
    kubectl get pod -l <app-label-from-deployment> -n <namespace> -w --timeout=3m
    ```
* **Analysis:** The old `Pending` pod will be terminated. I should see a new pod go from `Pending` -> `ContainerCreating` -> `Running`.

---

## Report the Outcome

My final step is to report what happened.

* **Success Scenario:** "Resolved: The pod `web-backend-123` was stuck in `Pending` due to an incorrect `nodeSelector` (was `disktype: ssd`). I have patched the parent deployment `web-backend` to use the correct label (`cloud.google.com/gke-local-ssd: "true"`). A new, healthy pod is now `Running`."
* **Failure/Escalation Scenario:** "Escalation: The pod `web-backend-123` is stuck. I attempted to fix its `nodeSelector`, but the new pod is also failing. **This requires manual investigation**."