# Voting App DevOps Project | AWS · Kubernetes · CI/CD · GitOps

A multi-repository voting application built to demonstrate cloud infrastructure, containerization, Kubernetes operations, automated delivery, GitOps deployment, security controls, and observability.

## The 5 service repositories

The application is split into five independent repositories. **This repository is the GitOps control repository** that contains the Argo CD Application definitions for those services.

| Repository | Responsibility | Delivery approach |
|---|---|---|
| [**voting-app-vote**](https://github.com/Mohammed-Azam-Sohail/voting-app-vote) | Python/Flask voting UI; accepts votes and places them on Redis. Implements Blue-Green deployment for the Vote service. | GitHub Actions |
| [**voting-app-result**](https://github.com/Mohammed-Azam-Sohail/voting-app-result) | Node.js result UI; reads processed voting results from PostgreSQL. | GitHub Actions |
| [**voting-app-worker**](https://github.com/Mohammed-Azam-Sohail/voting-app-worker) | .NET background worker; consumes queued votes from Redis and writes processed results to PostgreSQL. | Jenkins and Docker Buildx |
| [**voting-app-redis**](https://github.com/Mohammed-Azam-Sohail/voting-app-redis) | Internal Redis queue between Vote and Worker. | Argo CD applies Kubernetes manifests |
| [**voting-app-db**](https://github.com/Mohammed-Azam-Sohail/voting-app-db) | PostgreSQL persistence for processed voting data. | Argo CD applies Kubernetes manifests |

```text
Voting App
├── voting-app-vote     # Vote UI and Blue-Green releases
├── voting-app-result   # Results UI
├── voting-app-worker   # Background vote processing
├── voting-app-redis    # Queue
└── voting-app-db       # PostgreSQL database

voting-app-gitops       # This repository: Argo CD Application definitions
```

## Project overview

The five services run as Kubernetes workloads on an AWS EC2-hosted, single-node `kubeadm` cluster. NGINX Ingress provides application entry points. CI pipelines build and publish container images, update the desired image references in Git, and Argo CD reconciles the Kubernetes manifests from each service repository.

The project focuses on operational practices used in DevOps and platform engineering: reproducible container builds, declarative deployments, health checks, resource governance, controlled releases, security controls, monitoring, logging, and troubleshooting.

## Architecture and request flow

```text
                         User / Browser
                               |
                          NGINX Ingress
                         /             \
                        v               v
                  Vote Service      Result Service
                        |               |
                        v               v
              +----------------+    PostgreSQL
              |                |
              v                v
          vote-blue         vote-green
              \                /
               \-- One color --/
                 selected by
                 vote Service
                        |
                        v
                      Redis
                    (vote queue)
                        |
                        v
                      Worker
                        |
                        v
                    PostgreSQL
```

**Vote processing:** the Vote service writes a vote message to Redis. The Worker consumes queued messages and stores processed voting data in PostgreSQL. The Result service reads the stored results from PostgreSQL. Vote does not write directly to PostgreSQL.

## What this GitOps repository manages

This repository contains the Argo CD `Application` manifests that connect the cluster to the five service repositories.

| File | Purpose |
|---|---|
| `vote-application.yaml` | Registers the Vote repository with Argo CD |
| `result-application.yaml` | Registers the Result repository with Argo CD |
| `worker-application.yaml` | Registers the Worker repository with Argo CD; includes an ignore rule for the HPA-managed replica field |
| `redis-application.yaml` | Registers the Redis repository with Argo CD |
| `db-application.yaml` | Registers the PostgreSQL repository with Argo CD |
| `apps/voting-app/namespace.yaml` | Defines the `voting-app` namespace |


## CI/CD and GitOps delivery

```text
Developer push
     |
     v
Service repository
     |
     +--> GitHub Actions (Vote and Result)
     |       - Run configured checks/tests
     |       - Build container image
     |       - Publish image to Docker Hub
     |       - Update desired image reference in Git
     |
     +--> Jenkins (Worker)
             - Validate/build with Docker Buildx
             - Publish image to Docker Hub
             - Update desired image reference in Git
                     |
                     v
              Argo CD watches the service repo's k8s/ path
                     |
                     v
              Kubernetes reconciles desired state
```

Argo CD is the deployment authority: CI updates Git and does not directly deploy the Worker with `kubectl`. The Vote pipeline includes code checks and security scanning with CodeQL and Trivy where configured. Pipeline details are maintained in the corresponding service repositories, as the workflows differ by service.

## Kubernetes, security, and reliability

The project uses Kubernetes features to make workloads more observable and controlled:

- Deployments and Services for the five components, with separate Blue and Green Deployments for Vote.
- NGINX Ingress for application routing.
- Liveness and readiness probes.
- CPU and memory requests and limits.
- Horizontal Pod Autoscalers (HPA), PodDisruptionBudgets (PDB), and a namespace ResourceQuota.
- ServiceAccounts, RBAC, NetworkPolicies, Secrets/ConfigMaps where configured, disabled automatic ServiceAccount token mounting, and non-root container execution.
- Trivy image scanning and CodeQL source analysis in the workflows where configured.


## Monitoring, metrics, and logging

- **Prometheus** collects application and cluster metrics.
- **Grafana** provides dashboards and visualization.
- **Alertmanager** handles configured alert routing.
- **Metrics Server** supplies resource metrics used by `kubectl top` and HPA.
- **Grafana Alloy → Loki → Grafana** provides centralized Kubernetes workload/application log collection and querying.

## Initial setup and useful commands

The namespace must exist before syncing the child Applications because their sync options include `CreateNamespace=false`. From the root of this repository, the child Application objects can be created or updated individually:

```bash
kubectl apply -f apps/voting-app/namespace.yaml
kubectl apply -f vote-application.yaml
kubectl apply -f result-application.yaml
kubectl apply -f worker-application.yaml
kubectl apply -f redis-application.yaml
kubectl apply -f db-application.yaml
```

Check Application status:

```bash
kubectl get applications -n argocd
kubectl describe application voting-app-vote -n argocd
kubectl describe application voting-app-worker -n argocd
```
