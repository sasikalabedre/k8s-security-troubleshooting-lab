# Topic 14: Kubernetes DNS Resolution and Troubleshooting

## Objective
Understand Kubernetes DNS, troubleshoot NXDOMAIN errors, and verify communication between a client Pod, Service, and backend NGINX Pod.

## Flow Diagram

Client Pod: dns-failure-demo
        |
        v
CoreDNS: 10.96.0.10
        |
        v
Resolve Service DNS name
        |
        v
Service: dns-test-service
        |
        v
Backend Pod: dns-backend
        |
        v
NGINX page returned

## 1. Create the DNS Test Pod

The file dns-failure-pod.yaml contains a restricted-security-compliant BusyBox Pod.

    kubectl apply -f dns-failure-pod.yaml
    kubectl get pod dns-failure-demo

Expected status: Running

## 2. Test an Invalid DNS Name

    kubectl exec dns-failure-demo -- nslookup invalid-service.test

Expected result: NXDOMAIN or a DNS lookup failure.

Meaning: The requested DNS name could not be found. This tests a nonexistent name; it does not prove CoreDNS is broken.

## 3. Check the Pod DNS Configuration

    kubectl exec dns-failure-demo -- cat /etc/resolv.conf

Important fields:
- nameserver 10.96.0.10: Kubernetes DNS Service IP
- search domains: Used to complete shorter DNS names
- ndots:5: Controls when search domains are tried

## 4. Verify Kubernetes DNS

    kubectl exec dns-failure-demo -- nslookup kubernetes.default.svc.cluster.local

Expected result: The name resolves to 10.96.0.1.

This confirms DNS works for the built-in Kubernetes Service name.

## 5. Create the DNS Test Service

The file dns-test-service.yaml defines a Service named dns-test-service with selector app: dns-test and port 80 targeting port 8080.

    kubectl apply -f dns-test-service.yaml
    kubectl get service dns-test-service

## 6. Check DNS Resolution for the Test Service

    kubectl exec dns-failure-demo -- nslookup dns-test-service.k8s-hardening.svc.cluster.local

Expected result: The DNS name resolves to the Service ClusterIP, even if the Service initially has no backend Pods.

Important: A Service can have a DNS record without having endpoints.

## 7. Check Service Endpoints

    kubectl get endpoints dns-test-service

Initially, the result may show no endpoints because no Pod matches the selector app: dns-test.

## 8. Create the Backend NGINX Pod

The file dns-backend-pod.yaml creates the Pod dns-backend with label app: dns-test.

    kubectl apply -f dns-backend-pod.yaml
    kubectl get pod dns-backend -o wide

Wait until the Pod status is Running.

## 9. Verify the Backend Endpoint

    kubectl get endpoints dns-test-service

Expected result: The endpoint shows the backend Pod IP and port 8080.

The Pod label app: dns-test matches the Service selector.

## 10. Test the NGINX Page Through the Service

    kubectl exec dns-failure-demo -- wget -qO- http://dns-test-service

Expected result: NGINX HTML content.

This tests the complete path from the client Pod through DNS and the Service to the backend NGINX Pod.

## 11. Test the Backend Pod IP Directly

First find the actual Pod IP:

    kubectl get pod dns-backend -o wide

Then, from a network that can reach the Pod IP, test port 8080:

    curl http://<BACKEND_POD_IP>:8080

Replace <BACKEND_POD_IP> with the actual IP from the previous command. Direct access from the host depends on the Kubernetes networking setup.

## Troubleshooting Flow

DNS lookup returns NXDOMAIN
        |
        v
Check the exact Service name and namespace
        |
        v
Check whether the Service exists
        |
        v
Test kubernetes.default.svc.cluster.local
        |
        v
If built-in DNS works, investigate the requested name
        |
        v
Check Service endpoints separately
        |
        v
If endpoints are missing, compare Service selector and Pod labels
        |
        v
Retest DNS and HTTP connectivity

## Key Lessons

- DNS translates a Service name into its ClusterIP.
- CoreDNS commonly uses the Service IP 10.96.0.10 in this lab.
- NXDOMAIN means the requested DNS name could not be found.
- A wrong or nonexistent Service name can cause NXDOMAIN.
- A Service with no backend endpoints can still resolve through DNS.
- Service selectors must match backend Pod labels.
- DNS resolution and application connectivity are separate checks.
- Use nslookup for DNS, kubectl get endpoints for backends, and wget or curl for HTTP testing.

## Cleanup

Delete the lab resources when you no longer need them:

    kubectl delete pod dns-failure-demo --ignore-not-found
    kubectl delete pod dns-backend --ignore-not-found
    kubectl delete service dns-test-service --ignore-not-found


