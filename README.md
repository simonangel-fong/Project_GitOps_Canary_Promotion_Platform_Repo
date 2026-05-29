# GitOps Risk Control

**Validate Early. Release Gradually. Detect Fast.**

> A production-style GitOps project that reduces release risk across the delivery lifecycle.
> It validates changes through `multi-repo` and `multi-environment promotion`, releases gradually with `canary deployment`, and detects post-release issues through **monitoring** and **alerting**.

![Git](https://img.shields.io/badge/git-%23F05033.svg?style=for-the-badge&logo=git&logoColor=white&style=plastic) ![Argo CD](https://img.shields.io/badge/Argo%20CD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white&style=plastic) ![Argo Rollouts](https://img.shields.io/badge/Argo%20Rollouts-EF7B4D?style=for-the-badge&logo=argo&logoColor=white&style=plastic) ![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-2088FF?style=for-the-badge&logo=githubactions&logoColor=white&style=plastic) ![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white&style=plastic) ![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white&style=plastic) ![Alertmanager](https://img.shields.io/badge/Alertmanager-E6522C?style=for-the-badge&logo=prometheus&logoColor=white&style=plastic) ![Slack](https://custom-icon-badges.demolab.com/badge/Slack-4A154B?logo=slack&logoColor=fff) <br>
![AWS](https://img.shields.io/badge/AWS-FF9900?style=for-the-badge&logo=amazonwebservices&logoColor=white&style=plastic) ![Amazon EKS](https://img.shields.io/badge/Amazon%20EKS-FF9900?style=for-the-badge&logo=amazoneks&logoColor=white&style=plastic) ![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white&style=plastic) ![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white&style=plastic) ![Kustomize](https://img.shields.io/badge/Kustomize-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white&style=plastic) ![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white&style=plastic) ![Spring Boot](https://img.shields.io/badge/Spring%20Boot-6DB33F?logo=springboot&logoColor=fff&style=plastic) ![React](https://img.shields.io/badge/React-%2320232a.svg?logo=react&logoColor=%2361DAFB&style=plastic) <br>

- [GitOps Risk Control](#gitops-risk-control)
  - [Challenge and Solution](#challenge-and-solution)
  - [Project Architecture](#project-architecture)
  - [Pre-release Risk Control](#pre-release-risk-control)
    - [Dedicated Repositories](#dedicated-repositories)
    - [Environment Strategy](#environment-strategy)
    - [Automated Promotion Pipeline](#automated-promotion-pipeline)
  - [Release Risk Control](#release-risk-control)
    - [Canary Demo: Happy Path Promotion](#canary-demo-happy-path-promotion)
    - [Canary Demo: Failure Rollback](#canary-demo-failure-rollback)
  - [Post-release Risk Control](#post-release-risk-control)
    - [OOM Incident Simulation](#oom-incident-simulation)
  - [Summary](#summary)

---

## Challenge and Solution

**Challenge:**

Every production release carries the risk of introducing **bugs that affect users and disrupt business operations**. <br>
How can changes be **validated early**, **released gradually**, and **monitored continuously** to reduce business impact?

**Solution:**

This project implements a **GitOps-based release risk control workflow** across three phases:

| Phase        | Project Approach                                                                                           | Goal                                              |
| ------------ | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| Pre-release  | Separate responsibilities with dedicated repositories and isolated `dev`, `stage`, and `prod` environments | Catch issues early before they reach production   |
| Release      | Use `canary deployment` and automated rollout control                                                      | Limit the impact of failed releases               |
| Post-release | Monitor system health and trigger alerts after deployment                                                  | Detect incidents quickly and reduce recovery time |

---

## Project Architecture

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
                  +-------------------------+-------------------------+
                  ^                         ^                         ^
                  |                         |                         |
             Provisioning            Container Image         GitOps Sync / Rollout
                  |                         |                         |
        +---------------------+  +---------------------+   +---------------------+
        |Infrastructure Repo  |  | Application Repo    |   | Platform Repo       |
        |                     |  |                     |   |                     |
        | Terraform           |  |App source code      |   | GitOps manifests    |
        | AWS / EKS clusters  |  | Docker build        |   | App-of-Apps         |
        | ArgoCD install      |  |  CI pipeline        |   | Add-ons / apps      |
        +---------------------+  +---------------------+   +---------------------+
                  ^                         ^                         ^
                  |                         |                         |
             Cloud Engineer              Developer            Platform Engineer

```

---

## Pre-release Risk Control

`Pre-release risk control` focuses on reducing coordination issues, catching problems early, and preventing unvalidated changes from reaching production.

### Dedicated Repositories

- The project separates **application**, **infrastructure**, and **platform** responsibilities into dedicated repositories.
- This separation creates clear **ownership boundaries**, reduces **coordination risk**, and makes changes **easier to review and audit**.

| Repository                                                                                      | Role                | Main Responsibility                                                                                                |
| ----------------------------------------------------------------------------------------------- | ------------------- | ------------------------------------------------------------------------------------------------------------------ |
| [Platform](https://github.com/simonangel-fong/Project_GitOps_Risk_Control_Platform_Repo.git)    | _Platform Engineer_ | Kubernetes add-ons, application manifests, sync waves, canary rollout configuration, monitoring, and notifications |
| [Application](https://github.com/simonangel-fong/Project_GitOps_Risk_Control_App_Repo.git)      | _Software Engineer_ | Application source code, Docker image build, image push, and deployment update trigger                             |
| [Infrastructure](https://github.com/simonangel-fong/Project_GitOps_Risk_Control_Infra_Repo.git) | _Cloud Engineer_    | AWS infrastructure, EKS clusters, Argo CD installation, and networking foundation                                  |

---

### Environment Strategy

- The project separates `dev`, `stage`, and `prod` environments using dedicated `Git branches`, `EKS clusters`, and `Kustomize overlays`.
- This enables a controlled promotion path where changes can be validated progressively before reaching production.

| Environment | Branch  | Cluster        | Manifest Path     | Purpose                                   | Characteristics                        |
| ----------- | ------- | -------------- | ----------------- | ----------------------------------------- | -------------------------------------- |
| `dev`       | `dev`   | `gitops-dev`   | `overlays/dev/`   | Early validation for development changes  | Fast-changing and flexible             |
| `stage`     | `stage` | `gitops-stage` | `overlays/stage/` | Production-like validation before release | Test-heavy and production-like         |
| `prod`      | `prod`  | `gitops-prod`  | `overlays/prod/`  | Live environment for end users            | Stable, reliable, and security-focused |

---

### Automated Promotion Pipeline

- The `CI/CD pipeline` connects the separated **repositories** and **environments** into one controlled delivery flow.
- It validates changes, promotes manifests across environments, and keeps production promotion approval-based.

| Environment | Owner & Trigger                                           | Pipeline Responsibility                                                                                        |
| ----------- | --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `dev`       | _Platform Engineer_ commits manifest changes              | Run manifest validation, security checks, GitOps sync, smoke test, and failure notification                    |
| `stage`     | Auto-promotion after successful dev validation            | Promote manifests to stage, run GitOps sync, execute load test, and send validation result                     |
| `prod`      | _Release Owner_ reviews and approves production promotion | Promote manifests to prod, run GitOps sync, send release notification, and hand off to post-release monitoring |

![cicd-pipeline-diagram](./docs/assets/cicd_pipeline_diagram.png)

- The pipeline keeps `dev` and `stage` highly automated for fast validation,
- The `prod` requires **human release approval** to protect production stability.

---

## Release Risk Control

- `Release risk control` focuses on limiting production impact when a new version reaches users.
- Instead of replacing the stable version all at once, the project uses `canary deployment`, automated rollout analysis, and GitOps-based recovery to release changes gradually and safely.
  - **Canary Deployment**: shifts a small portion of production traffic to the new version before full rollout
  - **Automated Rollout Analysis**: evaluates rollout health before increasing traffic
  - **Rollback/revert Strategy**: stops unsafe releases and restores the stable version through GitOps-based recovery

```text
New Image / Manifest Update
        ↓
Argo CD Syncs Rollout Manifest
        ↓
Argo Rollouts Starts Canary Release
        ↓
Shift Limited Traffic to New Version
        ↓
Run AnalysisTemplate Checks
        ↓
+----------------------------+
↓                            ↓
Metrics Healthy              Metrics Failed
↓                            ↓
Promote New Version          Roll Back to Stable Version
↓                            ↓
Send Slack Notification      Send Slack Notification
```

---

### Canary Demo: Happy Path Promotion

A new version passes rollout analysis, traffic is gradually promoted, and `Slack` receives the deployment result.

![canary-happy-path](docs/assets/canary-happy-path.gif)

---

### Canary Demo: Failure Rollback

A bad release fails health or metric validation, `Argo Rollouts` rolls back to the stable version, and `Slack` receives the rollback notification.

![canary-db-failure-rollback](docs/assets/canary-db-failure-rollback.gif)

---

## Post-release Risk Control

`Post-release risk control` focuses on detecting production issues quickly after deployment.
Monitoring dashboards and incident alerts help identify abnormal behavior and support faster recovery.

- `Grafana dashboards`: **visualize** application and cluster health after deployment

  ![dashboard](./docs/assets/grafana_dashboard.png)

- `Alertmanager`: **notify** operators when abnormal conditions occur

  ![slack-alert](./docs/assets/slack_alert_post_deploy.png)

---

### OOM Incident Simulation

- Demo triggers a controlled post-release memory issue to demonstrate alerting, dashboard visibility, log investigation, and recovery behavior.

![oom-demo-alert](./docs/assets/oom_demo_alert.png)

![oom-demo-dashboard](./docs/assets/oom_demo_dashboard.png)

![oom-demo-log](./docs/assets/oom_demo_log.png)

![oom-demo-rollback](./docs/assets/oom_demo_rollback.png)

---

## Summary

| Phase        | Solution                      | Risk Reduced                   | Description                                       |
| ------------ | ----------------------------- | ------------------------------ | ------------------------------------------------- |
| Pre-release  | Separate repos                | Coordination risk across roles | Separates responsibilities with clearer ownership |
| Pre-release  | Environment isolation         | Production-readiness risk      | Isolates `dev`,`stage`,`prod` environments        |
| Pre-release  | CI/CD across repos & env      | Bug/security risk              | Catches issues before promotion                   |
| Release      | Canary deployment             | Rollout risk                   | Limits blast radius of failed releases            |
| Release      | Automated analysis / rollback | Manual response risk           | Stops unhealthy rollout faster                    |
| Post-release | Prometheus/Grafana monitoring | Visibility risk                | Shows system health after deployment              |
| Post-release | Alertmanager alerts           | Operation risk                 | Helps identify incidents sooner and reduce MTTR   |
