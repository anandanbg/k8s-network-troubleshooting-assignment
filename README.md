# Kubernetes Network Troubleshooting Assignment

## Environment & Architecture
- **Cluster:** Local Kind (Kubernetes in Docker) cluster
- **Namespace:** \support-lab\
- **Traffic Flow:** \client\ (curl pod) ──▶ \rontend-svc\ ──▶ \rontend\ (nginx:80) ──▶ \ackend-svc\ ──▶ \ackend\ (http-echo:8080)

## 1. Symptom
- Requests from \client\ to \http://frontend-svc\ succeeded (\HTTP/1.1 200 OK\).
- Requests from \rontend\ to \http://backend-svc\ failed with:
  \wget: can't connect to remote host (10.96.120.50): Connection refused\

## 2. Investigation
Ran layer-by-layer Kubernetes networking diagnostics inside the \support-lab\ namespace:
1. \kubectl get pods -n support-lab -o wide --show-labels\
2. \kubectl exec -n support-lab deploy/frontend -- nslookup backend-svc.support-lab.svc.cluster.local\
3. \kubectl get endpoints -n support-lab\
4. \kubectl describe svc backend-svc -n support-lab\

## 3. Observations
- **Pod Health:** All pods (\client\, \rontend\, \ackend\) were \1/1 Running\, and the \ackend\ pod carried the label \pp=backend\.
- **DNS Resolution:** \ackend-svc.support-lab.svc.cluster.local\ resolved cleanly to ClusterIP \10.96.120.50\, confirming CoreDNS was healthy.
- **Endpoints:** \kubectl get endpoints -n support-lab\ showed \ackend-svc\ had \<none>\ endpoints.
- **Service Selector:** \kubectl describe svc backend-svc -n support-lab\ revealed \Selector: app=backend-v2\.

## 4. Root Cause
The \ackend-svc\ Service label selector (\pp: backend-v2\) did not match the \ackend\ Deployment pod label (\pp: backend\). Because no pods matched the selector, Kubernetes did not attach any endpoints to \ackend-svc\, causing traffic to its ClusterIP to be refused.

## 5. Resolution
Reapplied the corrected \manifests/backend-svc.yml\ with \selector: app=backend\:
\\\ash
kubectl apply -f manifests/backend-svc.yml
\\\

## 6. Verification
- \kubectl get endpoints -n support-lab\ confirmed the \ackend\ pod IP and port \:8080\ were attached to \ackend-svc\.
- \kubectl exec -n support-lab deploy/frontend -- wget -qO- http://backend-svc\ returned \Hello from Backend!\, confirming end-to-end connectivity was restored.
