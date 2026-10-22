
cd ~/k8s-hardening/06-container-security

cat > README.md <<'EOF'
# Topic 6: Container Security Hardening

## Objective

Learn and practise three important Kubernetes container security controls:

1. Non-root execution
2. Read-only root filesystem
3. Linux capabilities

## Folder Structure

```text
06-container-security/
├── hardened-pod.yaml
└── README.md
```

## Flow Diagram

```text
Create Hardened Pod
        |
        v
Verify Non-Root Execution
        |
        v
Test Read-Only Root Filesystem
        |
        v
Verify Linux Capabilities
        |
        v
Verify Writable Volumes
        |
        v
Complete Security Checks
```

## 1. Apply the Pod

Apply the hardened Pod manifest:

```bash
kubectl apply -f hardened-pod.yaml
```

Check the Pod status:

```bash
kubectl get pods
```

Our Pod uses the `nginxinc/nginx-unprivileged:1.27` image.

## 2. Non-Root Execution

### Configuration

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 101
  runAsGroup: 101
```

### Verify

```bash
kubectl exec insecure-pod -- id
```

### Explanation

- UID `0` means root.
- Any non-zero UID means non-root.
- Our container runs as UID `101`, a non-root user.

Running as non-root reduces the privileges available to the application.

## 3. Read-Only Root Filesystem

### Configuration

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

### Test the root filesystem

```bash
kubectl exec insecure-pod -- sh -c 'touch /readonly-test'
```

Expected result:

```text
Read-only file system
```

### Explanation

The container cannot write to its root filesystem because it is configured as read-only.

## 4. Linux Capabilities

### Configuration

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

### Verify capabilities

```bash
kubectl exec insecure-pod -- sh -c 'grep -E "Cap(Inh|Prm|Eff|Bnd|Amb):" /proc/1/status'
```

### Explanation

- `CapPrm` represents permitted capabilities.
- `CapEff` represents effective capabilities.
- Zero values indicate that no capabilities are permitted or effective for the inspected process.

Dropping all capabilities reduces unnecessary privileges.

## 5. Volumes and Volume Mounts

### Configuration

```yaml
volumeMounts:
  - name: tmp
    mountPath: /tmp
  - name: cache
    mountPath: /var/cache/nginx
  - name: run
    mountPath: /var/run

volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
```

### Explanation

- `volumes` defines storage for the Pod.
- `volumeMounts` attaches storage to a directory inside the container.
- `emptyDir: {}` creates temporary storage associated with the Pod.

Writable volumes allow NGINX to write required temporary files while the root filesystem remains read-only.

### Test a writable volume

```bash
kubectl exec insecure-pod -- sh -c 'touch /tmp/test-file && echo SUCCESS'
```

Expected result:

```text
SUCCESS
```

The write succeeds because `/tmp` is a writable mounted volume.

## 6. Seccomp Profile

### Configuration

```yaml
seccompProfile:
  type: RuntimeDefault
```

### Explanation

This uses the container runtime's default seccomp profile to restrict certain system calls.

## 7. Complete Security Configuration

The Pod uses these security controls:

| Control | Purpose |
|---|---|
| `runAsNonRoot: true` | Requires non-root execution |
| `runAsUser: 101` | Specifies the container user |
| `allowPrivilegeEscalation: false` | Prevents privilege escalation |
| `readOnlyRootFilesystem: true` | Makes the root filesystem read-only |
| `capabilities.drop: [ALL]` | Drops Linux capabilities |
| `seccompProfile: RuntimeDefault` | Applies the runtime's default seccomp profile |
| `emptyDir: {}` | Provides temporary writable storage |

## 8. Cleanup

Delete the practice Pod when finished:

```bash
kubectl delete pod insecure-pod
