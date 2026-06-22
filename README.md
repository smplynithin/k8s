# DevShelf — End-to-End Cloud Native DevOps Project

## Project Overview

DevShelf is a full-stack developer bookmark and code snippet management application built to demonstrate an end-to-end DevOps lifecycle.

The project started as a Docker Compose application and was progressively transformed into a production-style Kubernetes deployment running on Amazon EKS.

The journey covers:

- Containerization using Docker
- Local Kubernetes deployment using Kind
- Kubernetes networking and security
- Persistent storage using StatefulSets and PVCs
- Monitoring using Prometheus and Grafana
- Helm package management
- Cloud deployment on AWS EKS
- Amazon ECR image management
- Dynamic storage using EBS CSI Driver
- AWS Load Balancer integration using ALB Controller
- IAM Roles for Service Accounts (IRSA)

---

# Application Architecture

The application consists of four components:

## Frontend

- React + Vite
- NGINX multi-stage Docker build
- Serves the web application

## Backend

- FastAPI
- REST APIs for bookmarks and snippets
- Health check endpoints
- Prometheus metrics exposure

## Database

- PostgreSQL 15
- StatefulSet deployment
- Persistent storage using PVC and AWS EBS gp3 volumes

## Cache

- Redis 7
- Used for caching search results and reducing database queries

---

# Evolution Journey

```
Docker Compose
       |
       |
Local Kubernetes (Kind)
       |
       |
Kubernetes Manifests
       |
       |
Security & Networking
       |
       |
Monitoring
(Prometheus + Grafana)
       |
       |
Helm Chart
       |
       |
AWS ECR
       |
       |
Amazon EKS
       |
       |
EBS CSI + IRSA
       |
       |
AWS ALB Ingress
```

---

# Final Production Architecture

```
                            Internet
                                |
                                |
                   AWS Application Load Balancer
                                |
                                |
                         Kubernetes Ingress
                                |
                 ---------------------------------
                 |                               |
          Frontend Service                Backend Service
             ClusterIP                      ClusterIP
                 |                               |
           React + NGINX Pods               FastAPI Pods
                                                    |
                                              Redis Cache
                                                    |
                                               PostgreSQL
                                                    |
                                                    PVC
                                                    |
                                              AWS EBS gp3
```

---

# Technology Stack

## Application

- React
- Vite
- NGINX
- FastAPI
- PostgreSQL
- Redis

---

## Containerization

- Docker
- Multi-stage Docker builds
- Docker Hub
- Amazon ECR

---

## Kubernetes

- Kind Cluster
- Pods
- Deployments
- StatefulSets
- Services
- Ingress
- ConfigMaps
- Secrets
- RBAC
- ServiceAccounts
- Network Policies
- Liveness and Readiness Probes
- Resource Requests and Limits
- Horizontal Pod Autoscaler (HPA)

---

## Packaging and Deployment

- Helm
- Helm Templates
- Environment-specific values

---

## Monitoring

- Prometheus
- Grafana
- ServiceMonitor
- Custom dashboards

Implemented metrics:

- Backend healthy replica count
- Request rate (RPS)
- CPU and Memory utilization

---

## AWS Cloud

- Amazon EKS
- Amazon ECR
- IAM
- IAM Roles for Service Accounts (IRSA)
- AWS Load Balancer Controller
- Application Load Balancer (ALB)
- Amazon EBS CSI Driver
- Amazon EBS gp3 volumes

---

# Major Engineering Challenges Solved

## Kubernetes Networking

### Issue

API requests through Ingress returned NGINX 404 errors.

### Root Cause

Incorrect host header and path rewriting behavior.

### Solution

- Validated traffic using `curl` with Host headers.
- Removed incorrect rewrite rules.
- Verified backend connectivity using port forwarding.

---

## Helm Template Debugging

### Issues

- Incorrect values structure.
- Image repository template mismatch.
- Existing cluster resource ownership conflicts.

### Solution

- Refactored `values.yaml`.
- Correctly separated cluster infrastructure from application resources.

---

## AWS Storage Challenges

### Issue

PostgreSQL failed to initialize on EBS volumes.

### Root Cause

Fresh EBS volumes contain a `lost+found` directory.

### Solution

Configured PostgreSQL:

```
PGDATA=/var/lib/postgresql/data/pgdata
```

---

## IAM and EKS Permissions

### Issue

EBS CSI Driver and AWS Load Balancer Controller failed due to missing permissions.

### Solution

Implemented:

- OIDC Provider
- IAM Roles
- Service Accounts
- IRSA architecture

---

# Project Repository Structure

```
k8s/
│
├── README.md                       # Overall project documentation
│
├── local-k8s/
│   └── README.md                   # Kind deployment journey
│
├── eks/
│   └── README.md                   # AWS EKS migration journey
│
├── manifests/
│   ├── backend/
│   ├── frontend/
│   ├── postgres/
│   ├── redis/
│   ├── ingress/
│   ├── secrets/
│   ├── configmap/
│   └── RBAC/
│
└── helm/
    └── devshelf/
```

---

# Key DevOps Concepts Demonstrated

This project demonstrates practical experience in:

- Containerizing applications
- Designing Kubernetes workloads
- Managing stateless and stateful applications
- Kubernetes networking and security
- Application observability
- Kubernetes package management
- Cloud-native storage
- AWS identity and access management
- Load balancing and ingress management
- Production troubleshooting

---

# Future Enhancements

Planned improvements:

- GitHub Actions CI/CD pipeline
- Automated Docker image builds
- Push images to Amazon ECR
- Automated Helm deployments to EKS
- Environment-based deployment strategy
- Slack notifications for deployment failures

---

# Conclusion

DevShelf demonstrates the complete journey of migrating an application from Docker Compose to a production-style Kubernetes platform on AWS.

The project involved not only deploying applications but also solving real-world operational challenges involving Kubernetes networking, Helm templating, AWS IAM, dynamic storage provisioning, and cloud-native infrastructure.

This repository represents practical hands-on experience across the complete DevOps lifecycle.
