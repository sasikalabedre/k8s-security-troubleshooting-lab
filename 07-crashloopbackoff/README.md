# Topic 7: CrashLoopBackOff Troubleshooting

## Objective
Learn how to reproduce, investigate, troubleshoot, and fix a Kubernetes CrashLoopBackOff.

## What Is CrashLoopBackOff?
CrashLoopBackOff occurs when a container repeatedly exits or crashes. Kubernetes attempts to restart the container but increases the delay between restart attempts.

CrashLoopBackOff is a symptom, not the root cause. The actual cause may be an application error, incorrect configuration, missing dependency, or an invalid startup command.

## Troubleshooting Flow Diagram

    Pod starts
        |
        v
    Container exits repeatedly
        |
        v
    Pod enters CrashLoopBackOff
        |
        v
    Check Pod status and restart count
        |
        v
    Check current and previous logs
        |
        v
    Describe Pod and inspect events
        |
        v
    Identify the root cause
        |
        v
    Fix the application or configuration
        |
        v
    Recreate the Pod if required
        |
        v
    Verify Pod health

## Step 1: Create the Intentionally Failing Pod

File: `crashloop-pod.yaml`

The initial container command was:

    command: ["sh", "-c", "echo 'Application starting'; sleep 5; exit 1"]

The command prints a startup message, waits five seconds, and exits with code 1. This deliberately simulates an application failure.

The Pod uses security settings that comply with the namespace's Restricted Pod Security Standard.

## Step 2: Apply the YAML

    kubectl apply -f crashloop-pod.yaml

Check the Pod:

    kubectl get pod crashloop-demo

The Pod may initially show Running or Error. After repeated failures, it should enter CrashLoopBackOff.

## Step 3: Check Container Logs

View the current or most recent container logs:

    kubectl logs crashloop-demo

View logs from the previous terminated container:

    kubectl logs crashloop-demo --previous

Expected log:

    Application starting

There is no error message because the command intentionally exits with code 1 without printing an error.

## Step 4: Describe the Pod

    kubectl describe pod crashloop-demo

Inspect these fields:
- State
- Last State
- Reason
- Exit Code
- Restart Count
- Events

Expected termination details:

    Last State: Terminated
    Reason: Error
    Exit Code: 1

Exit code 1 indicates that the process exited unsuccessfully. It does not, by itself, explain the full root cause.

## Step 5: Inspect Kubernetes Events

    kubectl get events --sort-by='.lastTimestamp'

Important events:

- `Scheduled`: Kubernetes assigned the Pod to a node.
- `Pulled`: The container image was downloaded or was already available.
- `Created`: The container was created.
- `Started`: The container started.
- `BackOff`: Kubernetes is delaying another restart because the container repeatedly failed.

Example event:

    Warning   BackOff   pod/crashloop-demo
    Back-off restarting failed container app

This confirms that Kubernetes is backing off from repeated restart attempts.

## Step 6: Fix the Failing Command

Open the manifest:

    nano crashloop-pod.yaml

Replace the failing command:

    command: ["sh", "-c", "echo 'Application starting'; sleep 5; exit 1"]

With the corrected command:

    command: ["sh", "-c", "echo 'Application starting'; sleep 3600"]

The container now stays running instead of intentionally exiting with code 1.

Save the file.

## Step 7: Recreate the Pod

A running Pod's container command cannot be changed in place. Delete and recreate the Pod using the updated manifest.

    kubectl delete pod crashloop-demo
    kubectl apply -f crashloop-pod.yaml

Check the status:

    kubectl get pod crashloop-demo

Check the logs:

    kubectl logs crashloop-demo

Expected:
- Status: `Running`
- Log: `Application starting`
- Restart count remains stable.

## Step 8: Complete Troubleshooting Command Reference

Check the Pod status:

    kubectl get pod crashloop-demo

Watch status and restart count:

    kubectl get pod crashloop-demo -w

Inspect current logs:

    kubectl logs crashloop-demo

Inspect previous container logs:

    kubectl logs crashloop-demo --previous

Inspect details and termination reason:

    kubectl describe pod crashloop-demo

Inspect recent events:

    kubectl get events --sort-by='.lastTimestamp'

## Key Lessons

- CrashLoopBackOff means a container repeatedly fails and Kubernetes is delaying its next restart.
- `kubectl logs` helps inspect application output.
- `kubectl logs --previous` is useful when a container has crashed and restarted.
- `kubectl describe pod` shows container states, exit codes, restart counts, and events.
- `kubectl get events` helps identify Kubernetes actions and restart backoff.
- Exit code 1 indicates an unsuccessful process exit, but further investigation is needed to identify the root cause.
- Fix the underlying application or configuration problem rather than just hiding the symptom.
- Keep security controls enabled while troubleshooting.
- Verify the Pod is healthy after applying the fix.

## Cleanup

Delete the practice Pod when you no longer need it:

    kubectl delete pod crashloop-demo


