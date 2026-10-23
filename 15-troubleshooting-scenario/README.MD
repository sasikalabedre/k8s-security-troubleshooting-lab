# Topic 15: Kubernetes Troubleshooting Decision Tree and Hardened Manifests

## 1. Objective

Learn a systematic method to troubleshoot Kubernetes workloads and verify that applications follow security best practices.

This topic covers:
- A Kubernetes troubleshooting decision tree
- Pod and Deployment health checks
- Service selector and endpoint troubleshooting
- DNS and connectivity checks
- Resource and scheduling checks
- Hardened Pod security settings
- A practical Service selector mismatch scenario

## 2. Troubleshooting Decision Tree

Start with the workload status and follow the matching branch.

User request
    |
    v
Check Pod status
    |
    +--> Pending
    |      |
    |      +--> Check events, node capacity, requests, taints and selectors
    |
    +--> ImagePullBackOff
    |      |
    |      +--> Check image name, tag, registry access and imagePullSecrets
    |
    +--> CrashLoopBackOff
    |      |
    |      +--> Check previous logs, application command and configuration
    |
    +--> OOMKilled
    |      |
    |      +--> Check memory limits and application memory usage
    |
    +--> CreateContainerConfigError
    |      |
    |      +--> Check referenced ConfigMaps, Secrets and configuration
    |
    +--> Running
           |
           v
       Check Service
           |
           +--> No endpoints
           |      |
           |      +--> Compare Service selector with Pod labels
           |
           +--> Endpoints exist
                  |
                  v
              Test DNS and connectivity
                  |
                  +--> DNS fails: check service name and CoreDNS
                  |
                  +--> HTTP fails: check targetPort, application and logs
                  |
                  +--> HTTP succeeds: verify the expected response

## 3. General Troubleshooting Commands

Check nodes:

    kubectl get nodes -o wide

Check Pods and their labels:

    kubectl get pods -o wide --show-labels

Check Deployments:

    kubectl get deployments

Check Services:

    kubectl get services

Check endpoints:

    kubectl get endpoints

Check recent events:

    kubectl get events --sort-by=.metadata.creationTimestamp

Describe a problematic Pod:

    kubectl describe pod POD_NAME

Read application logs:

    kubectl logs POD_NAME

Read logs from the previous crashed container:

    kubectl logs POD_NAME --previous

Check resource usage when metrics are available:

    kubectl top pods
    kubectl top nodes

## 4. Practical Scenario: Service Selector Mismatch

### Architecture

Client Pod
    |
    v
Service: troubleshooting-web-service
    |
    v
Service selector matches Pod labels
    |
    v
Endpoint
    |
    v
Hardened NGINX Pod on port 8080
    |
    v
HTTP 200

### Files

The manifest is stored in:

    service-mismatch.yaml

It defines:
- Deployment: troubleshooting-web
- Service: troubleshooting-web-service
- Container image: nginxinc/nginx-unprivileged:1.27
- Container port: 8080
- Pod namespace: k8s-hardening

### Step 1: Apply the manifest

    kubectl apply -f service-mismatch.yaml

The initial scenario intentionally used an incorrect Service selector:

    selector:
      app: wrong-label

The Deployment's Pod template used:

    labels:
      app: troubleshooting-web

### Step 2: Find the root cause

Check Pod labels:

    kubectl get pods --show-labels

Check the Service selector:

    kubectl describe service troubleshooting-web-service

Check Service endpoints:

    kubectl get endpoints troubleshooting-web-service

If the Service has no endpoints, compare its selector with the labels on the intended backend Pods.

### Step 3: Fix the selector

Update the Service selector in service-mismatch.yaml to:

    selector:
      app: troubleshooting-web

Apply the corrected manifest:

    kubectl apply -f service-mismatch.yaml

Verify that endpoints now exist:

    kubectl get endpoints troubleshooting-web-service

### Step 4: Test the Service

Create a compliant test Pod that sends an HTTP request:

    kubectl run service-check \
      --image=curlimages/curl:8.12.1 \
      --restart=Never \
      --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":10001,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"service-check","image":"curlimages/curl:8.12.1","command":["curl","-sS","-w","\nHTTP %{http_code}\n","http://troubleshooting-web-service"],"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}'

Check the test result:

    kubectl logs service-check

Expected result:

    HTTP 200

If the test Pod already exists, inspect its logs instead of creating it again. If the Pod has already completed and needs to be recreated, delete it first:

    kubectl delete pod service-check

Then repeat the test command.

## 5. Hardened Manifest Checklist

For production-oriented workloads, review these controls where compatible with the application:

- Use a non-root user.
- Set allowPrivilegeEscalation to false.
- Drop unnecessary Linux capabilities, preferably ALL.
- Set seccompProfile to RuntimeDefault.
- Use a read-only root filesystem when the application supports it.
- Mount writable directories explicitly when required.
- Define appropriate CPU and memory requests and limits.
- Use trusted images and pin image digests where appropriate.
- Store sensitive values in Kubernetes Secrets or an approved external secret manager.
- Apply least-privilege RBAC.
- Restrict network traffic with suitable NetworkPolicies.
- Configure readiness, liveness and startup probes as appropriate.

Example security settings:

    securityContext:
      runAsNonRoot: true
      runAsUser: 101
      seccompProfile:
        type: RuntimeDefault

    containers:
      - name: nginx
        image: nginxinc/nginx-unprivileged:1.27
        ports:
          - containerPort: 8080
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
              - ALL

These are example settings, not a complete production manifest. Validate image compatibility, writable paths, probes, resource settings and organization policies before deployment.

## 6. Root Cause and Resolution

Problem:
The Service selector did not match the labels on the Deployment's Pods.

Impact:
The Service had no matching endpoints, so it could not forward traffic to the intended backend.

Resolution:
Corrected the selector from app: wrong-label to app: troubleshooting-web, reapplied the manifest, verified endpoints, and tested the Service.

## 7. Key Learning

Troubleshoot from the outside in:

Workload status
    -> Pod events and logs
    -> Deployment configuration
    -> Service selector
    -> Endpoints
    -> DNS and network connectivity
    -> Application response

Always confirm the root cause with evidence before changing a manifest.

## 8. Cleanup

Delete the test Pod:

    kubectl delete pod service-check --ignore-not-found

Delete the Deployment and Service:

    kubectl delete -f service-mismatch.yaml

## 9. Completion Checklist

- [ ] Followed the troubleshooting decision tree.
- [ ] Checked Pod status, labels, events and logs.
- [ ] Identified the incorrect Service selector.
- [ ] Corrected the selector and verified endpoints.
- [ ] Tested the Service and confirmed HTTP 200.
- [ ] Reviewed hardened Pod security settings.
- [ ] Documented the root cause, fix and verification.

