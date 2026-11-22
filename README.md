Kubernetes Network Troubleshooting Assignment

Overview

This repository contains the Kubernetes manifests and Root Cause Analysis (RCA) for troubleshooting a Service-to-Pod connectivity failure in a local Kind (Kubernetes in Docker) cluster.

Architecture

Cluster: Local Kind cluster (imesh-lab)

Namespace: support-lab

Traffic Flow: client (curl pod) -> frontend-svc -> frontend (nginx:80) -> backend-svc -> backend (http-echo:8080)

Repository Structure

.
├── manifests/
│   ├── backend-deploy.yml
│   ├── backend-svc.yml
│   ├── client.yml
│   ├── frontend-deploy.yml
│   └── frontend-svc.yml
└── README.md


Symptom

What was failing?

Connectivity from client to frontend-svc worked normally (HTTP/1.1 200 OK).

Connectivity from frontend to backend-svc failed with a Connection refused error:

wget: can't connect to remote host (10.96.120.50): Connection refused


Investigation

What commands did you use?

# 1. Check pod status, IPs, and labels
kubectl get pods -n support-lab -o wide --show-labels

# 2. Verify DNS resolution from inside the frontend pod
kubectl exec -n support-lab deploy/frontend -- nslookup backend-svc.support-lab.svc.cluster.local

# 3. Check Service Endpoints
kubectl get endpoints -n support-lab

# 4. Inspect the backend-svc configuration
kubectl describe svc backend-svc -n support-lab


Observations

What did you discover during troubleshooting?

Pod Health: All pods (client, frontend, backend) were healthy and in 1/1 Running state. The backend pod had the label app=backend.

DNS Resolution: backend-svc.support-lab.svc.cluster.local resolved cleanly to ClusterIP 10.96.120.50, confirming CoreDNS was functioning properly.

Endpoints: kubectl get endpoints -n support-lab showed backend-svc had <none> endpoints, meaning no backend pod IP was mapped to the Service.

Service Selector: kubectl describe svc backend-svc -n support-lab revealed Selector: app=backend-v2.

Root Cause

What exactly caused the connectivity failure?
There was a Service selector mismatch. The backend-svc Service was configured with the selector app: backend-v2, whereas the backend Deployment pods were labeled app: backend. Because no pods matched the selector, Kubernetes did not attach any endpoints to backend-svc, causing traffic sent to its ClusterIP (10.96.120.50) to be refused.

Resolution

What did you change to fix it?
Updated the backend-svc Service selector from app: backend-v2 back to app: backend by reapplying the corrected manifest:

kubectl apply -f manifests/backend-svc.yml


Verification

How did you verify that the issue was resolved?

Confirmed that the backend pod IP and port :8080 were reattached to backend-svc:

kubectl get endpoints -n support-lab


Re-tested connectivity from frontend to backend-svc:

kubectl exec -n support-lab deploy/frontend -- wget -qO- http://backend-svc


Result: Returned Hello from Backend!, confirming end-to-end connectivity was restored.
