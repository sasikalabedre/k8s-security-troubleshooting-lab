# Topic 8: ImagePullBackOff

## Objective
Learn how to troubleshoot and fix Kubernetes image-pull failures.

## 1. Create an Image Pull Failure

Use an invalid image tag in `imagepullbackoff-pod.yaml`:

    image: nginx:invalid-tag-for-lab

Apply the manifest:

    kubectl apply -f imagepullbackoff-pod.yaml
    kubectl get pod imagepullbackoff-demo

Expected status: `ErrImagePull` or `ImagePullBackOff`.

## 2. Troubleshoot the Failure

Inspect the Pod details and events:

    kubectl describe pod imagepullbackoff-demo
    kubectl get events --sort-by='.lastTimestamp'

Check the Events section for the exact image-pull error.

## 3. Fix the Image

Update the image in `imagepullbackoff-pod.yaml`:

    image: nginxinc/nginx-unprivileged:1.27

Delete the failed Pod and create it again:

    kubectl delete pod imagepullbackoff-demo
    kubectl apply -f imagepullbackoff-pod.yaml
    kubectl get pod imagepullbackoff-demo

Expected status: `Running`.

## 4. Verify the Pod

    kubectl describe pod imagepullbackoff-demo
    kubectl logs imagepullbackoff-demo

Confirm that the Pod is running and the container starts successfully.

## 5. Image Tag vs Digest

### Image Tag

Example:

    nginxinc/nginx-unprivileged:1.27

- Easy to read and use.
- Tags can be changed to point to different image content.

### Image Digest

Example format:

    nginxinc/nginx-unprivileged@sha256:<actual-digest>

- Identifies specific image content.
- Helps make deployments reproducible.
- Always use a real, verified digest; the example above is a placeholder.

## 6. Troubleshooting Flow

    Pod fails to start
           |
           v
    kubectl describe pod
           |
           v
    Check the Events section
           |
           v
    Verify image name, tag, or digest
           |
           v
    Check registry authentication or network access if needed
           |
           v
    Correct the image reference
           |
           v
    Recreate the Pod
           |
           v
    Verify Pod status is Running

## 7. Common Causes

- Incorrect image name.
- Invalid or nonexistent image tag.
- Incorrect image digest.
- Private registry authentication failure.
- Registry or network connectivity problems.
- Rate limits imposed by the image registry.

## 8. Key Lessons

- `ErrImagePull` indicates an image-pull attempt failed.
- `ImagePullBackOff` means Kubernetes is delaying and retrying image pulls.
- `kubectl describe pod` is a key first troubleshooting command.
- Read the Events section to identify the actual cause.
- Correct the image reference or resolve registry access issues.
- Image digests pin specific image content.
- Verify the Pod status after applying the fix.

## Cleanup

    kubectl delete pod imagepullbackoff-demo


