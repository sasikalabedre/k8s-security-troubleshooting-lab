# Topic 11: CreateContainerConfigError

## Objective
Understand how missing configuration resources, such as Secrets, can prevent a Kubernetes container from starting.

## 1. Create the Failure

The file `config-error-pod.yaml` references a Secret named `missing-app-secret` and a key named `password`.

Apply the manifest:

    kubectl apply -f config-error-pod.yaml

Check the Pod:

    kubectl get pod config-error-demo

Inspect the error:

    kubectl describe pod config-error-demo

Expected error: `CreateContainerConfigError`.

## 2. Identify the Cause

The container references a Secret that does not exist.

The relevant configuration is:

    env:
      - name: APP_PASSWORD
        valueFrom:
          secretKeyRef:
            name: missing-app-secret
            key: password

Kubernetes cannot configure the container until the required Secret and key are available.

## 3. Fix the Missing Secret

Create the Secret with the required key:

    kubectl create secret generic missing-app-secret --from-literal=password='LabPass123!'

Check the Pod again:

    kubectl get pod config-error-demo

Expected status: `Running`.

Kubernetes automatically retries container setup. When the missing Secret becomes available, the existing Pod can recover without being deleted or reapplied.

## 4. Verify the Secret

    kubectl describe secret missing-app-secret

This displays the Secret metadata and key names without displaying the secret values.

## 5. Troubleshooting Flow

    Pod fails to start
            |
            v
    kubectl describe pod
            |
            v
    Check Events for missing Secret or ConfigMap
            |
            v
    Verify the resource name and required key
            |
            v
    Create or correct the missing dependency
            |
            v
    Kubernetes retries container setup
            |
            v
    Verify the Pod is Running

## 6. Other Possible Causes

- A referenced Secret does not exist.
- A referenced ConfigMap does not exist.
- A required key is missing from a Secret or ConfigMap.
- A volume references an unavailable configuration resource.
- The Pod configuration contains an invalid reference.

Use `kubectl describe pod` to identify the actual cause before changing anything.

## 7. Key Lessons

- `CreateContainerConfigError` indicates Kubernetes could not prepare the container configuration.
- Check the Pod Events section for the specific error.
- Verify referenced Secret and ConfigMap names and keys.
- Kubernetes can retry automatically after a missing dependency is created.
- You do not always need to delete and recreate a Pod to fix a missing Secret.
- Avoid placing real passwords directly in YAML files or Git repositories.

## Cleanup

    kubectl delete pod config-error-demo
    kubectl delete secret missing-app-secret

