# Kubernetes Secrets Handling and External Secret Management

## Objective

Learn how to create Kubernetes Secrets, consume them as environment variables and mounted files, harden Pods, and prevent credential files from being committed to Git.

## 1. Create a Kubernetes Secret

Create a dummy Secret for practice:

```bash
kubectl create secret generic app-credentials \
  --from-literal=username=demo-user \
  --from-literal=password=demo-pass-123
```

Verify the Secret without displaying its values:

```bash
kubectl get secrets
kubectl describe secret app-credentials
```

**Security:** Base64 encoding is not encryption. Never use real credentials in practice examples or commit them to Git.

## 2. Consume Secrets as Environment Variables

Manifest: `secret-env-pod.yaml`

The Pod reads the `username` and `password` keys from `app-credentials` using `secretKeyRef`.

Apply and verify
## 2. Consume Secrets as Environment Variables

Manifest: `secret-env-pod.yaml`

Apply and verify the Pod:

```bash
kubectl apply -f secret-env-pod.yaml
kubectl wait --for=condition=Ready pod/secret-env-pod --timeout=120s
kubectl exec secret-env-pod -- id
kubectl exec secret-env-pod -- sh -c 'printenv | cut -d= -f1 | grep "^APP_"'
```

Expected environment variable names:

- `APP_USERNAME`
- `APP_PASSWORD`

Do not print actual Secret values in logs or terminal output.

## 3. Consume Secrets as Mounted Files

Manifest: `secret-volume-pod.yaml`

Apply the Pod and wait until it is ready:

```bash
kubectl apply -f secret-volume-pod.yaml
kubectl wait --for=condition=Ready pod/secret-volume-pod --timeout=120s
```

List the mounted files without displaying their contents:

```bash
kubectl exec secret-volume-pod -- ls -ln /etc/app-secrets
```

Test readability:

```bash
kubectl exec secret-volume-pod -- sh -c '
test -r /etc/app-secrets/username &&
test -r /etc/app-secrets/password &&
echo "Both Secret files are readable"
'
```

## 4. Pod Security Context

The Secret-consuming Pods use a non-root user and the default seccomp profile.

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  runAsGroup: 10001
  fsGroup: 10001
  seccompProfile:
    type: RuntimeDefault
```

The container security context also disables privilege escalation and drops all Linux capabilities. The Secret-volume Pod uses a read-only root filesystem.

## 5. Secret Consumption Flow Diagram

```text
        Kubernetes Secret
          app-credentials
                 |
          +------+------+
          |             |
          v             v
 Environment variables  Secret volume
          |             |
          v             v
  secret-env-pod    secret-volume-pod
          |             |
          v             v
   Application      /etc/app-secrets/
                    username
                    password
```

## 6. Git Protection

The project-level `.gitignore` helps prevent common local credential files from being accidentally committed.

```gitignore
.env
.env.*
*.secret
*-secret.yaml
secrets/
*.pem
*.key
credentials.json
```

Verify the file and Git status:

```bash
cd ~/k8s-hardening
cat .gitignore
git status --short
```

Important: `.gitignore` does not remove files already tracked by Git. Never commit real credentials or tokens.

## 7. External Secret Management

For production environments, consider an external secret manager such as Google Secret Manager.

### Architecture Flow

```text
Google Secret Manager
          |
          v
External Secrets Operator
          |
          v
Kubernetes Secret
          |
          v
Application Pod
```

An External Secrets Operator can retrieve selected values from an external provider and synchronize them into Kubernetes Secrets.

Production security practices:

- Use workload identity or another secure authentication method.
- Apply least-privilege RBAC to Secret access.
- Configure Kubernetes Secret encryption at rest.
- Rotate credentials regularly.
- Avoid exposing credentials in logs or shell history.
- Monitor access to sensitive resources.

External secret managers improve credential storage and rotation workflows, but synchronized Kubernetes Secrets still require protection.

## 8. Cleanup Practice Resources

Run these commands only when you no longer need the practice Pods or Secret:

```bash
kubectl delete -f secret-volume-pod.yaml
kubectl delete -f secret-env-pod.yaml
kubectl delete secret app-credentials
```

## 9. Key Takeaways

- Kubernetes Secrets store sensitive configuration data.
- Base64 encoding is not encryption.
- Secrets can be consumed as environment variables or mounted files.
- Non-root execution, `fsGroup`, dropped capabilities, and read-only filesystems strengthen Pod security.
- RBAC and encryption at rest are important for protecting Secrets.
- `.gitignore` helps prevent accidental commits but is not a complete secret-protection mechanism.
- External secret managers can improve credential storage and rotation workflows.

EOF

tail -60 README.md
