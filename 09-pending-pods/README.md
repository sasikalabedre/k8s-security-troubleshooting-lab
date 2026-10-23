# Topic 9: Pending Pods

## Objective
Understand why a Kubernetes Pod remains Pending and how to troubleshoot scheduling failures caused by insufficient resources.

## 1. Create a Pod with Excessive Resource Requests

The `pending-pod.yaml` manifest requests 100 CPU cores and 100 GiB of memory.

Apply the manifest:

    kubectl apply -f pending-pod.yaml
    kubectl get pod pending-demo

Expected status: `Pending`.

## 2. Troubleshoot the Pod

Inspect the Pod details and scheduling events:

    kubectl describe pod pending-demo
    kubectl get events --sort-by='.lastTimestamp'

Check the Events section for a message such as:

    Insufficient cpu

The scheduler cannot place the Pod on a node that lacks the requested CPU capacity.

## 3. Fix the Resource Requests

Reduce the requests to realistic values:

    sed -i 's/cpu: "100"/cpu: "100m"/; s/memory: "100Gi"/memory: "64Mi"/' pending-pod.yaml

Delete and recreate the Pod:

    kubectl delete pod pending-demo
    kubectl apply -f pending-pod.yaml
    kubectl get pod pending-demo

Expected status: `Running`, provided the node has sufficient available resources.

## 4. Verify the Fix

    kubectl describe pod pending-demo
    kubectl get pod pending-demo -o wide

Confirm that the Pod is scheduled to a node and is running.

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
    Identify the scheduling reason
            |
            v
    Insufficient CPU or memory?
            |
            v
    Review and adjust resource requests
            |
            v
    Recreate the Pod
            |
            v
    Verify Running status

## 6. Other Possible Causes of Pending Pods

- Insufficient CPU or memory.
- Node selectors that match no nodes.
- Untolerated node taints.
- PersistentVolumeClaims that are not bound.
- Affinity or topology constraints that cannot be satisfied.

Use `kubectl describe pod` to identify the actual reason before changing the configuration.

## 7. Key Lessons

- `Pending` means the Pod has not completed scheduling or setup.
- Resource requests influence whether the scheduler can place a Pod on a node.
- CPU request `100m` means 100 millicores, or 0.1 CPU core.
- Memory request `64Mi` means 64 mebibytes.
- Resource requests are not the same as resource limits.
- Always check Pod events instead of guessing the cause.

## Cleanup

    kubectl delete pod pending-demo

