# 🏗️ Cloud-Native Enterprise Platform (CNEP)v2.0

> **A Senior-Level Reference Architecture for Secure, Scalable, and Automated Kubernetes Operations.**

Production-Grade, Zero-Trust Platform for Scalable Microservices (Zawatu API).

---

![alt text](https://img.shields.io/badge/Infrastructure-Terraform-623CE4?logo=terraform)
![alt text](https://img.shields.io/badge/Orchestration-GKE_Private-326CE5?logo=kubernetes)
![alt text](https://img.shields.io/badge/Secrets-HashiCorp_Vault-FFEC6E?logo=hashicorpvault)
![alt text](https://img.shields.io/badge/GitOps-ArgoCD-EF7B4D?logo=argo)
![alt text](https://img.shields.io/badge/Metrics-Prometheus-E6522C?logo=prometheus)

## 🎯 Executive Overview

This repository demonstrates a **Production-Ready Platform** designed to solve the "Day 2" challenges of Kubernetes. It implements a **Zero-Trust Security model**, **GitOps-driven deployments**, and **Self-Healing Infrastructure** using an industry-standard stack (GCP, Terraform, Vault, ArgoCD).

### Key Architectural Pillars

- **Infrastructure as Code (IaC):** Modular Terraform with remote state locking and multi-environment isolation.
- **Zero-Trust Security:** Runtime secret injection via HashiCorp Vault (No static secrets).
- **GitOps Delivery:** Pull-based continuous delivery using ArgoCD for 100% drift detection.
- **Full-Stack Observability:** LGTM Stack (Loki, Grafana, Tempo, Mimir) for deep-service tracing.
- **Governance:** Policy-as-Code enforcing resource limits and non-root execution.

---

## 🗺️ High-Level Architecture (The Big Picture)

code Mermaid

graph TD
subgraph "External Traffic"
DNS(Cloud DNS) --> WAF(Cloud Armor)
WAF --> GCLB(Global Load Balancer)
end

    subgraph "GKE Private Cluster (Internal VPC)"
        Ingress[Nginx Ingress Controller] --> App[Zawatu API - Canary/Stable]
        App --> Sidecar[Vault Agent Sidecar]
        App --> Mesh[Istio/Cilium eBPF]
    end

    subgraph "The Control Plane"
        Argo[ArgoCD] -- Pulls Config --> Git[(GitHub Repository)]
        Vault[(HashiCorp Vault)] -- Identity Auth --> App
        Prom[Prometheus] -- Scraping --> App
    end

    subgraph "Managed Services (GCP)"
        App -- Private Service Connect --> SQL[(Cloud SQL)]
        Vault -- Auto-Unseal --> KMS[Cloud KMS]
        Terraform -- Manages --> GKE[GKE Cluster]
    end

---

## 🛠️ The Tech Stack (Deep Dive)

Component Technology Role
Cloud Provider Google Cloud Platform (GCP) Core Infrastructure
Infrastructure Terraform (Modules) Declarative IaC with Remote State Locking
Orchestration GKE (Autopilot/Standard) Container Runtime & Scheduling
Secret Mgmt HashiCorp Vault Dynamic Secrets & Runtime Injection
Continuous Delivery ArgoCD GitOps Reconciliation & Drift Detection
Traffic Mgmt Nginx Ingress / Gateway API North-South Traffic Control
Observability Prometheus, Grafana, Loki The LGTM Stack (Metrics & Logs)
Security/Policy Kyverno Policy-as-Code Admission Controller
🚀 Deployment Guide (Step-by-Step)

1.  Phase 1: Pre-requisites & Auth

    Step 1.1: Initialize GCP Project and enable required APIs (Compute, GKE, KMS, Secret Manager).

    Step 1.2: Configure gcloud local authentication.

    Step 1.3: Create a GCS Bucket for Terraform Remote State.

2.  Phase 2: Infrastructure Provisioning (Day 0)

    Step 2.1: Navigate to infra/terraform/environments/prod.

    Step 2.2: Execute terraform init and terraform plan.

    Step 2.3: Execute terraform apply. This builds the VPC, Subnets, KMS Keys, and the Private GKE Cluster.

3.  Phase 3: Platform Bootstrapping (Day 1)

    Step 3.1: Connect to the cluster: gcloud container clusters get-credentials.

    Step 3.2: Install the ArgoCD Controller using Helm.

    Step 3.3: Apply the Root App (App-of-Apps): kubectl apply -f k8s/gitops/root-app.yaml.

        ArgoCD will now automatically provision Vault, Prometheus, and Ingress.

4.  Phase 4: Security Hardening (HashiCorp Vault)

    Step 4.1: Initialize Vault and store unseal keys in GCP KMS (Auto-unseal).

    Step 4.2: Enable Kubernetes Auth Method.

    Step 4.3: Configure Dynamic DB Secrets for the zawatu application.

5.  Phase 5: Application Deployment (Day 2)

    Step 5.1: Push code changes to services/zawatu-api.

    Step 5.2: GitHub Actions builds, scans (Trivy), and pushes image to Artifact Registry.

    Step 5.3: CI updates the GitOps repo. ArgoCD triggers a Canary Rollout.

---

## 🛡️ Security & Zero-Trust Architecture

    Runtime Secrets: Applications never see a long-lived password. They fetch short-lived tokens from Vault using Workload Identity.

    Network Isolation: All GKE nodes are private. Communication between namespaces is blocked by default via NetworkPolicies.

    Supply Chain: Images are signed with Cosign. The cluster uses an Admission Controller to block unsigned images.

    No SSH: Debugging is done via ephemeral containers or logged interactive sessions—never direct SSH to nodes.

---

## 📊 Observability & SRE (The 4 Golden Signals)

The platform is pre-configured with Grafana dashboards focusing on:

    Latency: p99 response time per endpoint.

    Traffic: Requests per second (Throughput).

    Errors: Error rate percentage (HTTP 5xx/4xx).

    Saturation: Memory and CPU utilization against defined limits.

🔄 Failure Scenarios & Self-Healing

    Scenario A (Pod Crash): K8s Liveness probes detect failure and restart the container in <10s.

    Scenario B (Node Failure): GKE auto-repair provisions a new node; the scheduler migrates workloads.

    Scenario C (Bad Deployment): Argo Rollouts detects an increase in 5xx errors from Prometheus and automatically rolls back to the previous stable version.

    Scenario D (Configuration Drift): If a user manually edits a resource, ArgoCD detects the diff and overwrites it to match Git within 3 minutes.

---

## 📂 The "Enterprise Platform" Directory Structure

```text
.
├── .github/                   # GitHub Actions Workflows (CI/CD)
│   └── workflows/             # Main build/test/deploy pipelines
│
├── infra/                     # Infrastructure as Code (Day 0)
│   └── terraform/             # Terraform Root
│       ├── modules/           # Reusable Modules (VPC, GKE, SQL, KMS)
│       └── environments/      # Environment-specific values
│           ├── dev/           # Development sandbox
│           └── prod/          # Production-grade infrastructure
│
├── k8s/                       # Kubernetes GitOps Manifests (Day 1)
│   ├── bootstrap/             # ArgoCD "App-of-Apps" entry point
│   ├── system/                # Cluster-wide Add-ons (The Platform)
│   │   ├── ingress-nginx/     # Traffic controller
│   │   ├── cert-manager/      # Automated SSL/TLS
│   │   ├── external-secrets/  # Vault to K8s sync
│   │   └── monitoring/        # Prometheus & Grafana stack
│   └── apps/                  # Business Logic (The Zawatu App)
│       ├── base/              # Common K8s manifests
│       └── overlays/          # Environment patches (Dev/Prod)
│
├── services/                  # Application Source Code
│   └── zawatu-api/            # The core Go/Node/Python API
│       ├── src/               # Application logic
│       ├── tests/             # Unit & Integration tests
│       └── Dockerfile         # Multi-stage secure build
│
├── security/                  # Security & Governance
│   ├── vault-policies/        # HCL files for Vault RBAC
│   └── policies/              # Kyverno/OPA Policy-as-Code rules
│
├── scripts/                   # Automation & Bootstrap
│   ├── setup-vault.sh         # Vault initialization logic
│   └── cluster-init.sh        # Local environment setup
│
└── docs/                      # High-level Documentation
    └── architecture/          # Diagrams and ADRs (Decision Records)

---

## 👨‍💻 Author & Lead Architect

| **Ndayishimiye Elvis** | **Senior Platform Engineer & Cloud-Native Architect** |
| :--- | :--- |
| 🚀 **Specialization** | Kubernetes Ecosystem (GKE/EKS), Zero-Trust Security (Vault), & GitOps (ArgoCD) |
| 🛠️ **Current Focus** | Engineering Resilient, Self-Healing Platforms for Global Scale |
| 🌐 **Portfolio** | [www.bindava.com](https://www.bindava.com) |
| 📧 **Contact** | [info@zawatu.com](mailto:info@zawatu.com) |

---

### 🔗 Connect With Me
[LinkedIn](https://www.linkedin.com/in/ndayishimiye-elvis) | [Portfolio](https://www.bindava.com) | [GitHub](https://github.com/elvisquant)

---

## 🤝 Contributing & Community

As a Lead Architect, I believe in the power of open-source collaboration and the "Platform Engineering" mindset.

    Contributions: Pull Requests (PRs) are welcome! If you find a bug or have a suggestion for improving the architecture, please open an issue first.

    Mentorship: If you are looking to scale your infrastructure or discuss Cloud-Native patterns, feel free to reach out via LinkedIn.

    Business Inquiries: For professional consulting or enterprise platform audits, contact me at info@zawatu.com.

---

## 🏆 Recognitions & Skills

![alt text](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![alt text](https://img.shields.io/badge/Google_Cloud-4285F4?style=for-the-badge&logo=google-cloud&logoColor=white)
![alt text](https://img.shields.io/badge/Vault-FFEC6E?style=for-the-badge&logo=hashicorp-vault&logoColor=black)
![alt text](https://img.shields.io/badge/Terraform-623CE4?style=for-the-badge&logo=terraform&logoColor=white)
![alt text](https://img.shields.io/badge/Argo_CD-EF7B4D?style=for-the-badge&logo=argo-cd&logoColor=white)

This platform is licensed under the MIT License - feel free to use it as a foundation for your enterprise journeys.
```
