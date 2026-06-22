# DevShelf Local Kubernetes Deployment (Kind Cluster)

## Project Overview

DevShelf is a full-stack developer bookmark and code snippet management application originally developed using Docker Compose and later migrated to Kubernetes.

This phase of the project focuses on understanding Kubernetes fundamentals by deploying the complete application stack on a local Kind (Kubernetes in Docker) cluster.

The goal was not only to deploy the application but also to understand how Docker concepts map to Kubernetes concepts.

---

# Application Architecture

The application consists of four containers:

1. Frontend
- React + Vite
- Served using Nginx

2. Backend
- FastAPI
- Provides REST APIs for bookmarks and snippets
- Exposes health and Prometheus metrics endpoints

3. Database
- PostgreSQL 15
- Stores bookmarks and snippets

4. Cache
- Redis 7
- Caches search results

---

# Docker Compose to Kubernetes Mapping

| Docker Concept | Kubernetes Equivalent |
|---------------|----------------------|
| Container | Pod |
| docker-compose.yml | Deployment / StatefulSet YAML |
| Docker Network | Services + Network Policies |
| Named Volume | PersistentVolumeClaim |
| Environment Variables | ConfigMaps & Secrets |
| healthcheck | Liveness & Readiness Probe |
| restart: always | Deployment ReplicaSet |
| Port Mapping | Service & Ingress |

---

# Local Kubernetes Environment

## Host Environment

- Windows 11 + WSL2 Ubuntu
- Docker Desktop
- Kind Kubernetes Cluster

## Cluster Details

Created a two-node Kubernetes cluster:

- Control Plane Node
- Worker Node

---

# Cluster Creation

## Install Kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv kind /usr/local/bin/
```

Verify:

```bash
kind --version
```

---

## Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/
```

Verify:

```bash
kubectl version --client
```

---

## Create Kind Cluster

```bash
kind create cluster --name devshelf
```

Verify:

```bash
kubectl get nodes
```

Expected:

```
NAME                      STATUS   ROLES
devshelf-control-plane    Ready    control-plane
devshelf-worker           Ready    <none>
```

---

# Namespace Creation

Created an isolated namespace:

```bash
kubectl apply -f namespace/devshelf-namespace.yml
```

Verify:

```bash
kubectl get namespaces
```

---

# Application Deployment

The resources were deployed in the following order:

## 1. ConfigMaps

Used for non-sensitive configuration.

Example:
- POSTGRES_USER
- POSTGRES_DB
- Environment values

Command:

```bash
kubectl apply -f configmap/
```

---

## 2. Secrets

Used for sensitive data.

Stored:

- Database URL
- Redis URL
- PostgreSQL Password

Command:

```bash
kubectl apply -f secrets/
```

---

## 3. PostgreSQL StatefulSet

PostgreSQL was deployed as a StatefulSet because it requires:

- Stable network identity
- Persistent storage

Resources:

- StatefulSet
- Headless Service
- PVC

Commands:

```bash
kubectl apply -f postgres/
kubectl apply -f pvc/
```

Verification:

```bash
kubectl get pods,pvc -n devshelf-prod
```

---

## 4. Redis Deployment

Redis was deployed as a Deployment because cache data is not critical.

Command:

```bash
kubectl apply -f redis/
```

---

## 5. Backend Deployment

Backend was configured with:

- Multiple replicas
- Secrets
- ConfigMaps
- Liveness Probe
- Readiness Probe

Command:

```bash
kubectl apply -f backend/
```

---

## 6. Frontend Deployment

Frontend was deployed with:

- Multiple replicas
- Nginx container

Command:

```bash
kubectl apply -f frontend/
```

---

# Kubernetes Networking

## Services

Created:

### Frontend Service

```
ClusterIP
Port 80
```

### Backend Service

```
ClusterIP
Port 80 → Container Port 8000
```

### PostgreSQL Service

```
Headless Service
```

### Redis Service

```
ClusterIP
```

---

# Ingress Controller Setup

Installed NGINX Ingress Controller.

Purpose:

- Single entry point into the cluster.
- Route traffic based on paths.

Routing:

```
http://devshelf.local/
                |
                |
          NGINX Ingress
              |
      ----------------
      |              |
 Frontend          Backend
                    |
                  /api
```

Ingress Rules:

```
/      → Frontend Service
/api   → Backend Service
```

---

# Major Challenges Faced

## Challenge 1: API returning NGINX 404

### Problem

Accessing:

```
/api/bookmarks
```

returned:

```
404 Not Found
nginx
```

---

### Root Cause

The request was not reaching the Kubernetes Ingress because the Host header was missing.

Ingress was configured with:

```
devshelf.local
```

---

### Solution

Tested using:

```bash
curl -H "Host: devshelf.local" \
http://localhost/api/bookmarks/
```

The request was correctly routed to the backend.

---

## Challenge 2: Incorrect Ingress Rewrite Rule

### Problem

Initial configuration used:

```
nginx.ingress.kubernetes.io/rewrite-target: /
```

which removed `/api` before sending the request to FastAPI.

Example:

```
Request:

/api/bookmarks

Rewritten to:

/bookmarks
```

FastAPI expected:

```
/api/bookmarks
```

---

### Solution

Removed the rewrite annotation and allowed the full path to reach the backend.

---

## Challenge 3: Backend Connectivity Debugging

Used port forwarding to isolate issues.

Command:

```bash
kubectl port-forward \
svc/devshelf-backend \
8000:80 \
-n devshelf-prod
```

Verified API:

```bash
curl localhost:8000/api/bookmarks
```

This confirmed the backend application was healthy.

---

## Security Implementation

Implemented:

- Kubernetes Secrets
- ConfigMaps
- Dedicated ServiceAccount
- Role
- RoleBinding
- Network Policies

Network Policy allowed:

```
Frontend Pods
       |
       |
Backend Pods
```

while restricting unnecessary communication.

---

# Health Monitoring

Implemented:

## Liveness Probe

Detects application failure and restarts the container.

---

## Readiness Probe

Controls whether a pod receives traffic from a Service.

---

# Scaling and Resource Management

Implemented:

- Horizontal Pod Autoscaler (HPA)
- CPU-based scaling using Metrics Server
- Resource Requests
- Resource Limits

Verification:

```bash
kubectl top pods
```

---

# Monitoring Setup

Installed:

- Prometheus
- Grafana
- kube-prometheus-stack Helm Chart

Implemented:

- ServiceMonitor for FastAPI metrics
- Custom Grafana Dashboard

Metrics created:

- Healthy Backend Replicas
- Request Rate (RPS)
- CPU and Memory Usage

---

# Useful Kubernetes Commands

## View Pods

```bash
kubectl get pods -A
```

## Describe Pod

```bash
kubectl describe pod <pod-name>
```

## View Logs

```bash
kubectl logs <pod-name>
```

## Execute Inside Pod

```bash
kubectl exec -it <pod-name> -- /bin/bash
```

## Check Services

```bash
kubectl get svc
```

## Check Ingress

```bash
kubectl get ingress
```

## Check Events

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

# Key Learnings

This migration provided hands-on experience with:

- Kubernetes architecture
- Pods and Deployments
- StatefulSets
- Persistent Volumes
- Services and DNS
- NGINX Ingress
- ConfigMaps and Secrets
- RBAC
- Network Policies
- Health Checks
- Horizontal Pod Autoscaling
- Monitoring using Prometheus and Grafana
- Debugging Kubernetes production-style issues

---

# Conclusion

The DevShelf application was successfully migrated from Docker Compose to a fully functional Kubernetes environment running on a local Kind cluster.

The project involved not only deployment but also troubleshooting real-world issues involving Ingress routing, path rewrites, networking, application health, and monitoring.
