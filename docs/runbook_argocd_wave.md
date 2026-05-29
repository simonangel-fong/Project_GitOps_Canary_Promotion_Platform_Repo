# Runbook - ArgoCD Sync Wave

[Back](../README.md)

- [Runbook - ArgoCD Sync Wave](#runbook---argocd-sync-wave)
  - [ArgoCD Sync Wave](#argocd-sync-wave)
    - [Numbering rules](#numbering-rules)
    - [Wave order](#wave-order)
  - [Cross-repo dependencies (debugging index)](#cross-repo-dependencies-debugging-index)

---

## ArgoCD Sync Wave

### Numbering rules

- Multiples of 100 for new top-level components.
- Adjacent integers (`N+1`) for configuration CRs that depend on a controller's CRDs.
- Leave the 100→app gap intact so cert-manager, OPA Gatekeeper, Kyverno, KEDA, etc. can land at 800–900 without renumbering apps.
- The `sync-wave` annotation is what ArgoCD reads; the filename prefix is for humans. Keep them aligned to avoid confusion.

- Numbering convention:
  - 0\*\*: ArgoCD configuration
  - 1\*\*: Cluster add-ons
  - 2\*\*: Netwoking add-ons
  - 5\*\*: middlewares
  - 9\*\*: Application

---

### Wave order

| Wave | Component                                         | Repo     | Purpose                                                      | Depends on                                                 |
| ---- | ------------------------------------------------- | -------- | ------------------------------------------------------------ | ---------------------------------------------------------- |
| 100  | Namespaces (`platform/namespaces/`)               | platform | Pre-creates workload namespaces                              | None                                                       |
| 110  | External Secrets Operator (Helm)                  | platform | Materialises Kubernetes Secrets from AWS SSM Parameter Store | IRSA role (infra), namespaces (wave 100)                   |
| 111  | ESO config — ClusterSecretStore + ExternalSecrets | platform | Defines the SSM connection and which parameters to sync      | ESO CRDs (wave 110)                                        |
| 112  | ArgoCD Notifications (Helm)                       | platform | Slack alerts on Application state changes                    | `argocd-notifications-secret` Secret (wave 110)            |
| 120  | Karpenter controller (Helm)                       | platform | Node autoscaling                                             | IRSA role + SQS interruption queue (infra)                 |
| 121  | Karpenter config — EC2NodeClass + NodePool        | platform | Tells Karpenter what nodes to provision                      | Karpenter CRDs (wave 121)                                  |
| 200  | AWS Load Balancer Controller (Helm)               | platform | Provisions ALBs/NLBs for Services and Gateways               | IRSA role (infra), Karpenter for capacity                  |
| 210  | Envoy Gateway controller (Helm)                   | platform | Ingress data-plane controller                                | ALBC (wave 200)                                            |
| 211  | Envoy GatewayClass + Gateway                      | platform | Concrete ingress endpoints                                   | Envoy CRDs (wave 210)                                      |
| 220  | External-DNS (Helm)                               | platform | Publishes Gateway hostnames to Cloudflare                    | Gateway (wave 220), `cloudflare-api-key` Secret (wave 110) |
| 500  | Argo Rollouts (Helm)                              | platform | Progressive delivery controller                              | None hard; precedes any app using Rollout CR               |
| 600  | kube-prometheus-stack (Helm)                      | platform | Prometheus & Grafana controller                              | None hard;                                                 |
| 900+ | User applications (backend, frontend, …)          | platform | Application workloads                                        | All platform pieces above                                  |

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

## Cross-repo dependencies (debugging index)

For each wave that needs something from the infra repo, this table lists exactly what the infra side must provide. When a wave is failing, start here: confirm every infra item on the row is present and correct before debugging the platform manifest.

| Wave | Platform (this repo)                 | Infra (Terraform repo)                                                                                             |
| ---- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| 110  | External Secrets Operator (Helm)     | ESO IAM role (IRSA, `ssm:GetParameter*` + optional `kms:Decrypt`)                                                  |
| 111  | ClusterSecretStore + ExternalSecrets | SSM parameters under the agreed path prefix (e.g. `/gitops/<env>/*`)                                               |
| 112  | ArgoCD Notifications (Helm)          | Slack token in SSM (wave 110)                                                                                      |
| 120  | Karpenter controller (Helm)          | Karpenter IAM role (IRSA), SQS interruption queue, EC2 instance profile, subnet/SG tags (`karpenter.sh/discovery`) |
| 121  | Karpenter NodePool + EC2NodeClass    | Same tags on subnets/SGs referenced in EC2NodeClass selectors                                                      |
| 200  | AWS Load Balancer Controller (Helm)  | ALBC IAM role (IRSA), VPC ID, subnet tags (`kubernetes.io/role/elb`, `kubernetes.io/role/internal-elb`)            |
| 210  | Envoy Gateway controller (Helm)      | None directly — relies on ALBC (wave 200) for LB provisioning                                                      |
| 211  | GatewayClass + Gateway               | None directly — ALB is created via wave 210's controller                                                           |
| 220  | External-DNS (Helm)                  | Cloudflare uses an API token from SSM (wave 111);                                                                  |
| 500  | Argo Rollouts (Helm)                 | Slack token in SSM (wave 110)                                                                                      |
| 900+ | User applications                    | Per-app SSM parameters (wave 111), any app-specific IAM roles (RDS access, S3 buckets, etc.)                       |

Waves not listed (or with "None") have no infra-repo dependency — failures there are platform-side only.
