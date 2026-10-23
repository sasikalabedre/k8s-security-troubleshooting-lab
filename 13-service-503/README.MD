# Topic 13: Kubernetes Service Returns 503

## Objective
Learn to troubleshoot a Service with no endpoints caused by an incorrect selector and restore connectivity.

## Flow Diagram

Deployment and Service
        |
        v
Check Service Endpoints
        |
        v
No Endpoints Found
        |
        v
Compare Pod Labels and Service Selector
        |
        v
Fix Service Selector
        |
        v
Apply Updated YAML
        |
        v
Test Service - HTTP 200

## 1. Apply the Lab

The service-503.yaml file contains a Deployment and a Service. The Service initially has an incorrect selector: app: wrong-label.

    kubectl apply -f service-503.yaml

## 2. Troubleshoot the Service

    kubectl get pods --show-labels
    kubectl get service web-503-service
    kubectl describe service web-503-service
    kubectl get endpoints web-503-service
    kubectl get endpointslices

Expected issue: the Service has no endpoints because its selector does not match the Pod label.

## 3. Fix the Selector

    sed -i 's/app: wrong-label/app: web-503-demo/' service-503.yaml
    kubectl apply -f service-503.yaml
    kubectl get endpoints web-503-service

Expected result: the endpoint shows the backend Pod IP and port 8080.

## 4. Test the Service

Create a test Pod that complies with Restricted Pod Security requirements.

    kubectl run service-test \
      --image=curlimages/curl:8.12.1 \
      --restart=Never \
      --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":10001,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"service-test","image":"curlimages/curl:8.12.1","command":["curl","-sS","-w","\nHTTP %{http_code}\n","http://web-503-service"],"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}'

Check the response:

    kubectl logs service-test

Expected result: HTTP 200

If the test Pod already exists, delete it before recreating it:

    kubectl delete pod service-test

Then rerun the kubectl run command above.

## 5. Troubleshooting Flow

Service not working
        |
        v
Check Service selector
        |
        v
Check Pod labels
        |
        v
Check endpoints
        |
        v
Correct selector mismatch
        |
        v
Apply YAML and retest

## Key Lessons

- Service selectors must match Pod labels exactly.
- A Service without matching Pods has no usable endpoints.
- An Ingress, proxy, or load balancer may return HTTP 503 when no healthy backends exist. A direct Service request may instead fail or time out.
- Use kubectl describe service and kubectl get endpoints to troubleshoot routing.
- Troubleshooting Pods must also comply with Pod Security requirements.

## Cleanup

    kubectl delete pod service-test --ignore-not-found
    kubectl delete -f service-503.yaml


