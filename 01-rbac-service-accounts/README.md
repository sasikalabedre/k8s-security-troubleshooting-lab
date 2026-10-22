
# Topic 1: RBAC, Service Accounts and Least Privilege

## Objective

Learn how Kubernetes controls access to cluster resources using Role-Based Access Control (RBAC), Service Accounts, Roles, and RoleBindings.

## 1. What Is RBAC?

RBAC stands for Role-Based Access Control.

It controls who can perform actions on Kubernetes resources.

For example:
- Allow a user or Service Account to view Pods.
- Prevent it from deleting Pods.
- Restrict permissions to a specific namespace.

## 2. Important Components

| Component | Purpose |
|---|---|
| ServiceAccount | Provides an identity for an application or workload |
| Role | Defines permissions within a namespace |
| RoleBinding | Assigns a Role to a user, group, or Service Account |
| Least privilege | Grants only the permissions required |

## 3. Flow Diagram

```text
ServiceAccount
      |
      v
RoleBinding
      |
      v
Role
      |
      v
Allowed Kubernetes API Actions
```

## 4. Create a ServiceAccount

Example manifest:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-reader
  namespace: k8s-hardening
automountServiceAccountToken: false
```

Save as `app-reader-serviceaccount.yaml` and apply:

```bash
kubectl apply -f app-reader-serviceaccount.yaml
```

Verify:

```bash
kubectl get serviceaccounts
```

`automountServiceAccountToken: false` disables automatic mounting of a Service Account API token into Pods using this account. It does not prevent a privileged administrator from obtaining credentials by other means.

## 5. Create a Role

Example manifest:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: k8s-hardening
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

Save as `pod-reader-role.yaml` and apply:

```bash
kubectl apply -f pod-reader-role.yaml
```

Verify:

```bash
kubectl get roles
kubectl describe role pod-reader
```

This Role allows reading Pods but does not grant permission to create or delete them.

## 6. Create a RoleBinding

Example manifest:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: k8s-hardening
subjects:
  - kind: ServiceAccount
    name: app-reader
    namespace: k8s-hardening
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
```

Save as `pod-reader-rolebinding.yaml` and apply:

```bash
kubectl apply -f pod-reader-rolebinding.yaml
```

Verify:

```bash
kubectl get rolebindings
kubectl describe rolebinding pod-reader-binding
```

## 7. Verify Permissions

Check whether the Service Account can read Pods:

```bash
kubectl auth can-i get pods --as=system:serviceaccount:k8s-hardening:app-reader
kubectl auth can-i list pods --as=system:serviceaccount:k8s-hardening:app-reader
kubectl auth can-i watch pods --as=system:serviceaccount:k8s-hardening:app-reader
```

Expected result for each command:

```text
yes
```

Check whether it can delete Pods or create Deployments:

```bash
kubectl auth can-i delete pods --as=system:serviceaccount:k8s-hardening:app-reader
kubectl auth can-i create deployments --as=system:serviceaccount:k8s-hardening:app-reader
```

Expected result:

```text
no
```

These commands check authorization permissions; they do not actually perform the requested actions.

## 8. Least Privilege Principle

Least privilege means granting only the permissions needed for an application to perform its job.

For example:

- A monitoring application may need to read Pods.
- It should not automatically receive permission to delete Pods.
- A namespace-scoped Role is preferable when cluster-wide access is unnecessary.

## 9. Troubleshooting

### Permission denied

Check the Role:

```bash
kubectl describe role pod-reader
```

Check the RoleBinding:

```bash
kubectl describe rolebinding pod-reader-binding
```

Check authorization:

```bash
kubectl auth can-i get pods --as=system:serviceaccount:k8s-hardening:app-reader
```

Ensure the RoleBinding references the correct Service Account and Role in the correct namespace.

## 10. Cleanup

Remove the practice resources when they are no longer needed:

```bash
kubectl delete rolebinding pod-reader-binding --ignore-not-found
kubectl delete role pod-reader --ignore-not-found
kubectl delete serviceaccount app-reader --ignore-not-found
```

## Result

Practised creating a Service Account, defining namespace-scoped permissions with a Role, assigning those permissions using a RoleBinding, and verifying allowed and denied actions.

EOF
