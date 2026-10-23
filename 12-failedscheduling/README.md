# Topic 12: FailedScheduling

## Objective
Understand why Kubernetes cannot schedule a Pod and how to troubleshoot and fix scheduling failures.

## 1. Create a Scheduling Failure

The `failed-scheduling-pod.yaml` manifest uses a node selector that does not match any node:

    nodeSelector:
      kubernetes.io/hostname: nonexistent-node

Apply the manifest:

    kubectl apply -f failed-scheduling-pod.yaml
    kubectl get pod failed-scheduling-demo

Expected status: `Pending`.

## 2. Troubleshoot the Pod

Inspect the Pod and its scheduling events:

    kubectl describe pod failed-scheduling-demo
    kubectl get events --sort-by='.lastTimestamp'

Look for an event similar to:

    0/2 nodes are available: node(s) didn't match Pod's node affinity/selector.

The scheduler cannot find a node matching the required hostname.

## 3. Fix the Node Selector

Check the available nodes:

    kubectl get nodes

Update the selector to use the existing node `minikube-m02`:

    sed -i 's/nonexistent-node/minikube-m02/' failed-scheduling-pod.yaml

Delete and recreate the Pod:

    kubectl delete pod failed-scheduling-demo
    kubectl apply -f failed-scheduling-pod.yaml
    kubectl get pod failed-scheduling-demo

Expected status: `Running`, provided the selected node is Ready and has sufficient resources.

## 4. Verify Scheduling

    kubectl get pod failed-scheduling-demo -o wide
    kubectl describe pod failed-scheduling-demo

Confirm that the Pod is assigned to `minikube-m02` and inspect its Events.

## 5. Troubleshooting Flow

    Pod remains Pending
            |
            v
    kubectl describe pod
            |
            v
    Check the Events section
            |
            v
    Identify the scheduling failure
            |
            v
    Verify node selector and node labels
            |
            v
    Correct the selector or other scheduling constraint
            |
            v
    Recreate the Pod if required
            |
            v
    Verify Pod status and assigned node

## 6. Other Causes of FailedScheduling

- Insufficient CPU or memory.
- Node selectors that match no nodes.
- Untolerated node taints.
- Affinity or topology constraints that cannot be satisfied.
- PersistentVolumeClaims that cannot be bound.

Always check the actual scheduler event before changing the configuration.

## 7. Key Lessons

- `FailedScheduling` means the scheduler cannot find a suitable node for the Pod.
- `Pending` is a Pod phase; `FailedScheduling` is commonly reported as a warning event.
- `nodeSelector` requires a node with matching labels.
- `kubectl get nodes --show-labels` displays node labels.
- `kubectl describe pod` helps identify the reason scheduling failed.
- Fix the actual scheduling constraint instead of guessing.

## Cleanup

    kubectl delete pod failed-scheduling-demo

