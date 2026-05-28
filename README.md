# GitOps Canary Promotion

**End to End. Auto-Promoted. Canary-Released**

> A production-style GitOps project that separates application, infrastructure, and platform delivery across 3 repositories. <br>
> It uses EKS, ArgoCD, Argo Rollouts, Terraform, and GitHub Actions to automate environment-based deployments, canary promotion, rollback, and post-deployment monitoring and alerting.

![Git](https://img.shields.io/badge/git-%23F05033.svg?style=for-the-badge&logo=git&logoColor=white&style=plastic) ![Argo CD](https://img.shields.io/badge/Argo%20CD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white&style=plastic) ![Argo Rollouts](https://img.shields.io/badge/Argo%20Rollouts-EF7B4D?style=for-the-badge&logo=argo&logoColor=white&style=plastic) ![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-2088FF?style=for-the-badge&logo=githubactions&logoColor=white&style=plastic) <br>
![AWS](https://img.shields.io/badge/AWS-FF9900?style=for-the-badge&logo=amazonwebservices&logoColor=white&style=plastic) ![Amazon EKS](https://img.shields.io/badge/Amazon%20EKS-FF9900?tyle=for-the-badge&logo=amazoneks&logoColor=white&style=plastic) ![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white&style=plastic) ![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white&style=plastic) ![Kustomize](https://img.shields.io/badge/Kustomize-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white&style=plastic) ![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white&style=plastic) <br>
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white&style=plastic) ![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white&style=plastic) ![Alertmanager](https://img.shields.io/badge/Alertmanager-E6522C?style=for-the-badge&logo=prometheus&logoColor=white&style=plastic) ![Slack](https://img.shields.io/badge/Slack-4A154B?style=for-the-badge&logo=slack&logoColor=white&style=plastic) <br>

- [GitOps Canary Promotion](#gitops-canary-promotion)
  - [1. Why This Project Exists](#1-why-this-project-exists)
    - [1.1 Managing App, Infrastructure, and Platform Changes](#11-managing-app-infrastructure-and-platform-changes)
    - [1.2 Releasing Safely Without Business Interruption](#12-releasing-safely-without-business-interruption)
  - [2. Project Architecture](#2-project-architecture)
  - [3. What This Platform Repo Manages](#3-what-this-platform-repo-manages)
    - [3.1 Canary Deployment and Auto Rollback](#31-canary-deployment-and-auto-rollback)
      - [Scenario Demo: Happy path promotion](#scenario-demo-happy-path-promotion)
      - [Scenario Demo: Failure rollback on DB connection error](#scenario-demo-failure-rollback-on-db-connection-error)
    - [3.2 Deployment Control with ArgoCD Sync Waves](#32-deployment-control-with-argocd-sync-waves)
    - [3.3 Automated CI/CD Pipelines and Promotion Flow](#33-automated-cicd-pipelines-and-promotion-flow)
    - [3.4 Environment Strategy](#34-environment-strategy)
  - [4. Operational Runbooks](#4-operational-runbooks)
  - [5. Limitations](#5-limitations)

---

## 1. Why This Project Exists

### 1.1 Managing App, Infrastructure, and Platform Changes

**Challenge:**

- In enterprise environments, _application code_, _cloud infrastructure_, and _Kubernetes platform configuration_ are often owned by different roles.
- Without clear separation and automated GitOps workflows, delivery can become slow, inconsistent, and difficult to audit.

**Solution:**

- This project uses a `3-repo GitOps strategy` to separate application, infrastructure, and platform responsibilities.
- `CI/CD pipelines` automate validation, image delivery, infrastructure provisioning, and manifest updates, making the delivery process more traceable and repeatable.

---

### 1.2 Releasing Safely Without Business Interruption

**Challenge:**

- Direct production releases increase the risk of downtime, failed deployments, and slow recovery.
- Without a controlled deployment strategy, issues may only be detected after they affect users.

**Solution:**

- This project implements `canary deployment` across isolated `dev`, `stage`, and `prod` environments.
- `Argo Rollouts`, automated analysis, monitoring, and rollback logic help detect issues early, reduce release risk, and protect business continuity.

---

## 2. Project Architecture

- 3-repo GitOps model to separate application delivery, infrastructure provisioning, and platform configuration.

```txt
                                      End Users
                                          |
                                          v
        +---------------------------------------------------------------------+
        |                            EKS Runtime                              |
        |                                                                     |
        |  Applications: Frontend App, Backend App                            |
        |                                                                     |
        |  Platform Add-ons: ESO, Karpenter, ALBC, Envoy, ExternalDNS         |
        |  Delivery & Observability: Argo Rollouts, Prometheus,               |
        |  Alertmanager, Slack Notifications                                  |
        +---------------------------------------------------------------------+
                                          ^
                                          |
                  +-------------------------+------------------------+
                  ^                         ^                        ^
                  |                         |                        |
             Provisioning            Container Image         GitOps Sync / Rollout
                  |                         |                        |
        +---------------------+  +---------------------+   +---------------------+
        |Infrastructure Repo  |  | Application Repo    |   | Platform Repo       |
        |                     |  |                     |   |                     |
        | Terraform           |  |App source code      |   | GitOps manifests    |
        | AWS / EKS clusters  |  | Docker build        |   | App-of-Apps         |
        | ArgoCD install      |  |  CI pipeline        |   | Add-ons / apps      |
        +---------------------+  +---------------------+   +---------------------+
                  ^                        ^                         ^
                  |                        |                         |
             Cloud Engineer             Developer            Platform Engineer

```

| Repository                                                                                                     | Main responsibility                                                                |
| -------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| [Platform Repository](https://github.com/simonangel-fong/Project_GitOps_Canary_Promotion_Platform_Repo.git)    | Add-ons, app manifests, sync waves, canary rollout, monitoring, Slack notification |
| [Application Repository](https://github.com/simonangel-fong/Project_GitOps_Canary_Promotion_App_Repo.git)      | Source code, Docker image build, image push, manifest/image update trigger         |
| [Infrastructure Repository](https://github.com/simonangel-fong/Project_GitOps_Canary_Promotion_Infra_Repo.git) | AWS, EKS clusters, ArgoCD installation, networking foundation                      |

---

## 3. What This Platform Repo Manages

This **platform repo** acts as the `GitOps control plane` for the project after the `EKS cluster` and `ArgoCD` are provisioned by the infrastructure repo.

It manages

- Kubernetes application manifests,
- ArgoCD App-of-Apps bootstrap resources,
- platform add-ons,
- environment overlays,
- canary rollout definitions,
- external secrets integration,
- monitoring configuration,
- and notification workflows.

The repository is organized around GitOps ownership, environment separation, and reusable automation:

```text
.
├── .github/
│   ├── actions/                 # Reusable GitHub Actions components
│   └── workflows/               # Platform validation and deployment workflows
│
├── apps/
│   ├── backend/                 # Backend Kubernetes manifests
│   │   ├── base/                # Shared backend manifests
│   │   └── overlays/
│   │       ├── dev/             # Dev-specific Kustomize configuration
│   │       ├── stage/           # Stage-specific Kustomize configuration
│   │       └── prod/            # Prod-specific Kustomize configuration
│   │
│   └── frontend/                # Frontend Kubernetes manifests
│       ├── base/                # Shared frontend manifests
│       └── overlays/
│           ├── dev/             # Dev-specific Kustomize configuration
│           ├── stage/           # Stage-specific Kustomize configuration
│           └── prod/            # Prod-specific Kustomize configuration
│
├── bootstrap/                   # ArgoCD App-of-Apps bootstrap manifests
│
├── platform/                    # Custom platform manifests and add-on configuration
│   ├── argocd-notifications/    # Slack notification templates and triggers
│   ├── external-secrets/        # External Secrets Operator deployment/configuration
│   ├── karpenter/               # Karpenter-related platform configuration
│   ├── envoy/                   # Envoy / ingress-related manifests
│   └── monitoring/              # Prometheus, Grafana, Alertmanager config
│
├── docs/                        # Operational runbooks and debugging notes
├── .checkov.yaml                # Policy-as-code configuration
└── README.md
```

---

### 3.1 Canary Deployment and Auto Rollback

`Canary deployment` reduces release risk by gradually shifting traffic to a new version, validating application health, and rolling back automatically before a bad release affects all users.

This project uses `Argo Rollouts`, `AnalysisTemplates`, `Prometheus metrics`, and `Slack notifications` to automate **canary promotion**, **failure detection**, **rollback**, and **release visibility**.

```text
New Image / Manifest Update
        v
ArgoCD Syncs Rollout Manifest
        v
Argo Rollouts Starts Canary Release
        v
Shift Small Portion of Traffic to New Version
        v
Run AnalysisTemplate Checks
        +----------------------------+
        v                            v
Metrics Healthy                Metrics Failed
        v                            v
Promote New Version            Roll Back to Stable Version
        v                            v
Send Slack Notification        Send Slack Notification
```

---

#### Scenario Demo: Happy path promotion

- A new version passes analysis, traffic is gradually promoted, and `Slack` receives the deployment result.

![canary-happy-path](docs/assets/canary-happy-path.gif)

---

#### Scenario Demo: Failure rollback on DB connection error

- A bad release fails health/metric validation, `Argo Rollouts` rolls back to the stable version, and `Slack` receives the rollback notification.

![canary-happy-path](docs/assets/canary-db-failure-rollback.gif)

For the full walkthrough, see [docs/canary_demo.md](docs/canary_demo.md).

---

### 3.2 Deployment Control with ArgoCD Sync Waves

Application resources depend on platform services such as secrets, ingress, DNS, rollout controllers, and monitoring, so deployment order must be controlled.

This repo uses `ArgoCD Sync Waves` to deploy resources layer by layer:

- lower wave numbers run first
- higher wave numbers run later.

ArgoCD Sync Wave Deployment Order:

```text
Wave 110  ── External Secrets Operator
              v
Wave 120  ── Karpenter
              v
Wave 200  ── AWS Load Balancer Controller
              v
Wave 210  ── Envoy Gateway
              v
Wave 220  ── ExternalDNS
              v
Wave 500  ── Argo Rollouts
              v
Wave 600  ── Prometheus Stack
              v
Wave 900  ── Backend Application
              v
Wave 910  ── Frontend Application
```

---

### 3.3 Automated CI/CD Pipelines and Promotion Flow

This project creates `automated CI/CD pipelines` to **validate platform changes**, **promote manifests across environments**, and **keep production releases approval-based**.

| Env     | Owner / Trigger                                           | Jobs Automated by Pipeline                                                                                          |
| ------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `dev`   | _Platform Engineer_ commits manifest changes              | Manifest lint check, security scan, GitOps sync, smoke test, failure notification                                   |
| `stage` | Auto-promotion after dev validation                       | Promote manifests to stage, GitOps sync, load test, validation notification                                         |
| `prod`  | _Release Owner_ reviews and approves production promotion | Promote manifests to prod, GitOps sync, release notification, post-deployment monitoring, Alertmanager notification |

- The pipeline keeps `dev` and `stage` highly automated for fast validation,
- The `prod` requires human release approval to protect production stability.

---

### 3.4 Environment Strategy

- This project **separates** `dev`, `stage`, and `prod` to support fast validation, production-like testing, and controlled production release.
- Each environment is protected by dedicated `git branch`, `cluster`, and `Kustomize configuration`, to ensure changes can move through the delivery flow **with lower risk**.

| Env     | Branch  | Dedicated Cluster | Manifest                            | Purpose                                            | Feature                            |
| ------- | ------- | ----------------- | ----------------------------------- | -------------------------------------------------- | ---------------------------------- |
| `dev`   | `dev`   | `gitops-dev`      | `overlays/dev/kustomization.yaml`   | Safe space for development and early validation    | Flexible, fast-changing            |
| `stage` | `stage` | `gitops-stage`    | `overlays/stage/kustomization.yaml` | Production-like environment for release validation | Test-heavy, production-like        |
| `prod`  | `prod`  | `gitops-prod`     | `overlays/prod/kustomization.yaml`  | Live environment for end users                     | Stable, reliable, security-focused |

---

## 4. Operational Runbooks

Runbooks for common platform issues and troubleshooting workflows:

- [Debugging `External DNS`](docs/debug_edns.md): Common ExternalDNS sync issues and troubleshooting steps
- [Debugging `AWS Load Balancer Controller`](docs/debug_lb.md): Common load balancer misconfigurations and fixes
- [Debugging `ArgoCD Sync Waves`](docs/argocd_wave.md): Sync wave ordering and dependency control
- [Debugging `Slack Notifications`](docs/debug_argo_notification.md): Slack notification setup and common issues for ArgoCD and Argo Rollouts

---

## 5. Limitations

This project focuses on GitOps delivery design, canary promotion, and platform automation. Some production-grade areas can be improved further:

- Separate clusters improve isolation but _increase operational cost and management overhead_.
  - **Improvement**: Add cost controls, shared platform modules, and automated cleanup for non-prod environments.
- Canary rollback depends on _selected health checks and metrics(pod readiness)_.
  - **Improvement**: Expand analysis with stronger SLO-based metrics such as error rate, latency, saturation, and business-level signals.
- Production promotion still requires _manual approval_.
  - **Improvement**: Integrate richer release evidence, approval history, and change-management records before production deployment.
- The project demonstrates platform automation but does not _cover full disaster recovery_.
  - **Improvement**: Add backup, restore, multi-AZ / multi-region failover testing, and recovery runbooks.
