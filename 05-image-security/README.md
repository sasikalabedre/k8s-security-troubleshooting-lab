
cat > README.md <<'EOF'
# Topic 5: Image Provenance, Scanning, Admission Controls and Audit Logs

## 1. Objective

This lab covers:
- Container image vulnerability scanning with Trivy
- Understanding vulnerability severity and unfixed findings
- Image digest pinning
- Running containers as non-root
- Enforcing a read-only root filesystem
- Kubernetes Pod Security Standards and admission controls
- Kubernetes API server audit logging

## 2. Lab Environment

- Kubernetes: Minikube
- Driver: Docker
- Nodes: 1 control plane and 1 worker
- Namespace: `k8s-hardening`
- Image scanner: Trivy
- Test image: `nginx:1.27`
- Hardened image: `nginxinc/nginx-unprivileged:1.27`

Check the environment:

```bash
minikube status
minikube profile list
kubectl get nodes -o wide
kubectl config view --minify -o jsonpath='{..namespace}{"\n"}'
```

## 3. Flow Diagram

```text
Container Image
      |
      v
Trivy Vulnerability Scan
      |
      v
Review HIGH / CRITICAL Findings
      |
      v
Choose a Suitable Image
      |
      v
Pin Image by Digest
      |
      v
Apply Container Security Settings
      |
      v
Kubernetes Admission Controls
      |
      v
Audit API Requests and Responses
```

## 4. Install Trivy on Ubuntu

Install the required packages:

```bash
sudo apt-get update
sudo apt-get install -y wget gnupg
```

Add the Trivy repository:

```bash
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" \
  | sudo tee /etc/apt/sources.list.d/trivy.list

sudo apt-get update
sudo apt-get install -y trivy
```

Verify installation:

```bash
trivy --version
```

## 5. Scan a Container Image

Run a complete scan:

```bash
trivy image nginx:1.27
```

Show only HIGH and CRITICAL findings:

```bash
trivy image --severity HIGH,CRITICAL nginx:1.27
```

Show HIGH and CRITICAL findings while ignoring unfixed vulnerabilities:

```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed nginx:1.27
```

Save reports:

```bash
trivy image --format json --output nginx-scan-report.json nginx:1.27

trivy image --format table --output nginx-scan-report.txt nginx:1.27
```

Check generated files:

```bash
ls -lh nginx-scan-report.json nginx-scan-report.txt
```

### Understanding the results

- `CRITICAL`: Highest severity; investigate before deployment.
- `HIGH`: Significant risk; review and remediate.
- `MEDIUM` and `LOW`: Review according to your risk policy.
- `--ignore-unfixed`: Hides findings without an available fix; it does not make an image safe.
- A scan is a point-in-time assessment. Rescan images regularly.

A vulnerability scan is evidence for a deployment decision, not a guarantee that an image is completely secure.

## 6. Image Provenance and Digest Pinning

Tags such as `nginx:1.27` are convenient, but tags can be moved to reference different image content.

A digest identifies specific image content. Pinning by digest helps make deployments reproducible.

Pull the hardened image:

```bash
docker pull nginxinc/nginx-unprivileged:1.27
```

Inspect its repository digest:

```bash
docker image inspect nginxinc/nginx-unprivileged:1.27 \
  --format '{{index .RepoDigests 0}}'
```

Inspect the image details:

```bash
docker image inspect nginxinc/nginx-unprivileged:1.27
```

The digest used in this lab was:

```text
nginxinc/nginx-unprivileged@sha256:f9dfa9c20b2b0b7c5cc830374f22f23dee3f750b6c5291ca7e0330b5c88e6403
```

Always verify the digest against the registry and the image you intend to deploy. A digest pins image content but does not, by itself, prove who built or signed it.

### Provenance verification in production

For stronger supply-chain security, consider:
- Verifying image signatures with Cosign.
- Checking build provenance and attestations.
- Using trusted registries and controlled build pipelines.
- Enforcing signature verification through an admission policy or webhook.

## 7. Deploy a Digest-Pinned Hardened Pod

The manifest `digest-pinned-pod.yaml` uses:
- A digest-pinned, unprivileged NGINX image
- UID and GID `101`
- `allowPrivilegeEscalation: false`
- A read-only root filesystem
- No Linux capabilities
- Writable `emptyDir` mounts for required runtime paths
- The `RuntimeDefault` seccomp profile

Apply the manifest:

```bash
kubectl apply -f digest-pinned-pod.yaml
```

Check the Pod:

```bash
kubectl get pod digest-pinned-pod
kubectl describe pod digest-pinned-pod
```

Verify the deployed image:

```bash
kubectl get pod digest-pinned-pod \
  -o jsonpath='{.spec.containers[0].image}{"\n"}'
```

Verify the container identity:

```bash
kubectl exec digest-pinned-pod -- id
```

Expected identity for this image:

```text
uid=101(nginx) gid=101(nginx) groups=101(nginx)
```

Verify container security settings:

```bash
kubectl get pod digest-pinned-pod \
  -o jsonpath='{.spec.containers[0].securityContext}{"\n"}'
```

Test whether the root filesystem is read-only:

```bash
kubectl exec digest-pinned-pod -- sh -c 'touch /readonly-test'
```

Expected result: `Read-only file system`. This is a successful security test because the write was blocked.

## 8. Admission Controls and Pod Security Standards

The namespace is configured to enforce the Restricted Pod Security Standard.

Check the labels:

```bash
kubectl get namespace k8s-hardening --show-labels
```

The relevant labels include:

```text
pod-security.kubernetes.io/enforce=restricted
pod-security.kubernetes.io/enforce-version=latest
pod-security.kubernetes.io/warn=restricted
pod-security.kubernetes.io/warn-version=latest
```

### Test an insecure Pod

The negative-test manifest `insecure-admission-pod.yaml` deliberately uses:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: insecure-admission-pod
  namespace: k8s-hardening
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      securityContext:
        privileged: true
```

Attempt to apply it:

```bash
kubectl apply -f insecure-admission-pod.yaml
```

Expected result: the Restricted Pod Security Standard rejects the Pod because privileged containers are not allowed.

Do not weaken the namespace policy just to make this test Pod run.

### Admission control flow

```text
kubectl apply
     |
     v
Kubernetes API Server
     |
     v
Authentication and Authorization
     |
     v
Admission Controls
     |
     +---- Policy violation ----> Reject request
     |
     +---- Policy passes -------> Continue processing
```

Admission control and audit logging are different:
- Admission controls decide whether an API request can proceed.
- Audit logging records API activity for investigation.

## 9. Kubernetes Audit Logging

Audit logging records API activity such as resource creation, updates, and deletion. Audit events can contain the requesting identity, action, resource, source IP, response status, and timestamps.

### Create the audit policy

The lab file `audit-policy.yaml` contains:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    verbs:
      - create
      - update
      - patch
      - delete
    resources:
      - group: ""
        resources:
          - pods
          - secrets
          - configmaps
          - services
      - group: apps
        resources:
          - deployments
          - replicasets

  - level: Metadata
    resources:
      - group: ""
        resources:
          - namespaces

  - level: None
    users:
      - system:kube-proxy
    verbs:
      - watch
```

`Metadata` records request metadata, not the full request or response bodies. The policy above records selected write operations; it does not record ordinary Pod list requests.

### Prepare the policy for Minikube

```bash
mkdir -p ~/.minikube/files/etc/ssl/certs

cp audit-policy.yaml \
  ~/.minikube/files/etc/ssl/certs/audit-policy.yaml
```

### Enable audit logging in Minikube

**This restarts the local cluster and may temporarily interrupt workloads.**

```bash
minikube stop

minikube start \
  --extra-config=apiserver.audit-policy-file=/etc/ssl/certs/audit-policy.yaml \
  --extra-config=apiserver.audit-log-path=-
```

The audit log path `-` directs audit output to the API server's standard output, which can be inspected through container logs.

### Verify the API server flags

```bash
kubectl get pod kube-apiserver-minikube -n kube-system \
  -o jsonpath='{.spec.containers[0].command}{"\n"}' \
  | grep -oE -- '--audit-policy-file=[^ ]+|--audit-log-path=[^ ]+'
```

Expected flags:

```text
--audit-policy-file=/etc/ssl/certs/audit-policy.yaml
--audit-log-path=-
```

### Read audit events

```bash
kubectl logs kube-apiserver-minikube -n kube-system \
  | grep 'audit.k8s.io/v1' \
  | tail -n 5
```

### Generate a test audit event

Create a ConfigMap in the default namespace:

```bash
kubectl create configmap audit-demo \
  --namespace=default \
  --from-literal=purpose=audit-test
```

Find the creation event:

```bash
kubectl logs kube-apiserver-minikube -n kube-system \
  | grep '"verb":"create"' \
  | grep '"resource":"configmaps"' \
  | tail -n 1
```

The event should include:
- `verb: create`
- `objectRef.name: audit-demo`
- `objectRef.namespace: default`
- `user.username`
- `responseStatus.code: 201`

Remove the test resource:

```bash
kubectl delete configmap audit-demo \
  --namespace=default \
  --ignore-not-found=true
```

### Audit logging flow

```text
kubectl command or API client
            |
            v
     Kubernetes API Server
            |
            v
      Audit Policy Rules
            |
            v
       Audit Event
            |
            v
 API Server Container Logs
            |
            v
 Review and Investigation
```

**Production note:** Sending audit logs to container stdout is useful for this local lab. Production clusters should use an appropriate audit backend, centralized log collection, retention controls, access restrictions, and monitoring. Audit logs may contain sensitive metadata and must be protected.

## 10. Troubleshooting

### Trivy command not found

```bash
trivy --version
sudo apt-get update
sudo apt-get install -y trivy
```

### Pod is rejected by admission controls

```bash
kubectl describe namespace k8s-hardening
kubectl get namespace k8s-hardening --show-labels
```

Review the error and correct the manifest rather than disabling security enforcement.

### Digest-pinned Pod is not starting

```bash
kubectl describe pod digest-pinned-pod
kubectl get events --sort-by=.metadata.creationTimestamp
```

Check the image digest, image availability, volume mounts, and container security settings.

### Audit events are not appearing

```bash
kubectl get pod kube-apiserver-minikube -n kube-system
kubectl get pod kube-apiserver-minikube -n kube-system \
  -o jsonpath='{.spec.containers[0].command}{"\n"}'

kubectl logs kube-apiserver-minikube -n kube-system \
  | grep 'audit.k8s.io/v1' \
  | tail -n 5
```

Check that the policy file exists at the configured path inside the control-plane node, that the API server accepted the audit flags, and that the tested operation matches the audit policy.

## 11. Cleanup

Remove the practice Pod:

```bash
kubectl delete pod digest-pinned-pod --ignore-not-found=true
```

Remove the negative-test Pod if one exists:

```bash
kubectl delete pod insecure-admission-pod --ignore-not-found=true
```

Remove the audit test ConfigMap:

```bash
kubectl delete configmap audit-demo \
  --namespace=default \
  --ignore-not-found=true
```

Restore the intended default namespace:

```bash
kubectl config set-context --current --namespace=k8s-hardening
```

Check remaining resources:

```bash
kubectl get pods
kubectl get events --sort-by=.metadata.creationTimestamp
```

Keep `audit-policy.yaml` and the scan reports if you want them as lab evidence. Scan reports can become outdated, so regenerate them when performing a new image assessment.

## 12. Key Takeaways

1. Scan container images before deployment.
2. Review vulnerability severity and available fixes.
3. Prefer trusted images and verify their provenance.
4. Pin images by digest for reproducible deployments.
5. Run containers as non-root and drop unnecessary capabilities.
6. Use read-only root filesystems and explicitly mount required writable paths.
7. Enforce Pod Security Standards through namespace admission controls.
8. Test that insecure manifests are rejected.
9. Configure Kubernetes audit policies to record relevant API activity.
10. Protect, retain, and monitor audit logs appropriately.

**Topic 5 completed:** Image scanning, digest pinning, container hardening, admission-control testing, and audit-log verification.

