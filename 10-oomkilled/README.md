# Topic 10: OOMKilled

## Objective
Understand why Kubernetes terminates a container due to excessive memory usage and how to troubleshoot the issue.

## 1. Create the Test Pod

The file `oomkilled-pod.yaml` creates a Pod with a memory limit of 32 MiB.

Apply the manifest:

    kubectl apply -f oomkilled-pod.yaml

Check the Pod status:

    kubectl get pod oomkilled-demo

## 2. Understand the Memory Settings

    resources:
      requests:
        memory: "16Mi"
      limits:
        memory: "32Mi"

- `requests.memory: 16Mi` — memory requested for Kubernetes scheduling decisions.
- `limits.memory: 32Mi` — the container's memory limit.
- `--vm-bytes 100M` — tells the stress program to attempt to allocate about 100 MB.

**Important:** 32 MiB is approximately 33.55 MB. The stress program attempts to allocate about 100 MB, exceeding the container's memory limit.

## 3. Understand the Container Specification

- `apiVersion: v1` — uses the Kubernetes core API.
- `kind: Pod` — creates a Pod.
- `metadata` — defines the Pod name and namespace.
- `restartPolicy: Never` — does not automatically restart a terminated container.
- `runAsNonRoot: true` — requires a non-root user.
- `runAsUser: 10001` — runs the process as user ID 10001.
- `seccompProfile: RuntimeDefault` — applies the runtime's default syscall security profile.
- `image: polinux/stress` — provides the stress-testing program.
- `args` — configures the memory stress test.
- `allowPrivilegeEscalation: false` — prevents privilege escalation.
- `capabilities.drop: [ALL]` — removes Linux capabilities from the container.

## 4. Troubleshooting Commands

Inspect the Pod:

    kubectl describe pod oomkilled-demo

Check the container's current or last termination state:

    kubectl get pod oomkilled-demo -o jsonpath='{.status.containerStatuses[0].state}{"\n"}{.status.containerStatuses[0].lastState}{"\n"}'

Check events:

    kubectl get events --sort-by='.lastTimestamp'

Check logs:

    kubectl logs oomkilled-demo

Look for `OOMKilled` in the termination reason.

## 5. Expected Result

The successful test produced:

- Termination reason: `OOMKilled`
- Exit code: `137`

Exit code 137 indicates the process was killed with SIGKILL. Combined with the `OOMKilled` reason, it confirms the memory-limit failure in this lab.

## 6. Troubleshooting Flow

    Create Pod with low memory limit
                 |
                 v
    Stress program attempts allocation
                 |
                 v
    Requested allocation exceeds limit
                 |
                 v
    Container is terminated
                 |
                 v
    Check kubectl describe pod
                 |
                 v
    Confirm OOMKilled and exit code 137

## 7. Linux Capabilities

Linux capabilities are individual permissions that allow a process to perform specific privileged actions.

Examples:
- `NET_BIND_SERVICE` — bind to ports below 1024.
- `NET_ADMIN` — perform certain network administration tasks.
- `CHOWN` — change file ownership.

The following configuration drops all Linux capabilities:

    securityContext:
      capabilities:
        drop:
          - ALL

This reduces the privileges available to the container.

## 8. Key Lessons

- `OOMKilled` means the container was terminated due to an out-of-memory condition.
- Memory requests help Kubernetes schedule Pods.
- Memory limits constrain container memory usage.
- 1 MiB = 1,048,576 bytes.
- 1 MB = 1,000,000 bytes.
- `kubectl describe pod` helps identify termination reasons and events.
- Exit code 137 alone does not prove an OOM event; check the termination reason.
- Non-root execution, seccomp, disabled privilege escalation, and dropping capabilities improve container security.

## Cleanup

    kubectl delete pod oomkilled-demo
