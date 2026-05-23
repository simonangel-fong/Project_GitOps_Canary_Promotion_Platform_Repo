# ArgoCD Sync Wave Plan

Numbering convention: gaps of 10 between critical waves so new components can be inserted without renumbering. Sub-waves (e.g. `11`, `31`) attach configuration CRs immediately after the controller that owns their CRDs.

## Wave order

| Wave | Component                                         | Repo     | Purpose                                                      | Depends on                                              |
| ---- | ------------------------------------------------- | -------- | ------------------------------------------------------------ | ------------------------------------------------------- |
| 1    | Namespaces (`platform/namespaces/`)               | platform | Pre-creates workload namespaces so wave-6 ExternalSecrets can write into them before the consuming chart installs | None                                                    |
| 5    | External Secrets Operator (Helm)                  | platform | Materialises Kubernetes Secrets from AWS SSM Parameter Store | IRSA role (infra), namespaces (wave 1)                  |
| 6    | ESO config — ClusterSecretStore + ExternalSecrets | platform | Defines the SSM connection and which parameters to sync      | ESO CRDs (wave 5)                                       |
| 10   | Karpenter controller (Helm)                       | platform | Node autoscaling                                             | IRSA role + SQS interruption queue (infra)              |
| 11   | Karpenter config — EC2NodeClass + NodePool        | platform | Tells Karpenter what nodes to provision                      | Karpenter CRDs (wave 10)                                |
| 20   | AWS Load Balancer Controller (Helm)               | platform | Provisions ALBs/NLBs for Services and Gateways               | IRSA role (infra), Karpenter for capacity               |
| 30   | Envoy Gateway controller (Helm)                   | platform | Ingress data-plane controller                                | ALBC (wave 20)                                          |
| 31   | Envoy GatewayClass + Gateway                      | platform | Concrete ingress endpoints                                   | Envoy CRDs (wave 30)                                    |
| 40   | External-DNS (Helm)                               | platform | Publishes Gateway hostnames to Cloudflare                    | Gateway (wave 31), `cloudflare-api-key` Secret (wave 6) |
| 70   | ArgoCD Notifications (Helm) — **deferred**        | platform | Slack alerts on Application state changes                    | `argocd-notifications-secret` Secret (wave 6)           |
| 90   | Argo Rollouts (Helm)                              | platform | Progressive delivery controller                              | None hard; precedes any app using Rollout CR            |
| 100+ | User applications (backend, frontend, …)          | platform | Application workloads                                        | All platform pieces above                               |

## Repo split

- **infra repo (Terraform)**: EKS cluster, VPC, IAM roles (ALBC, Karpenter, ESO), SQS interruption queue, OIDC provider, ArgoCD bootstrap, app-of-apps.
- **platform repo (this one, GitOps)**: every Helm chart, CRD, and CR listed above.
- **Handoff between the two**: IRSA role ARNs are pasted into the relevant Helm `values` block. ARNs are not secrets — committing them to git is the standard pattern.

## Why ESO is first

ESO produces Secrets that downstream waves consume (`cloudflare-api-key` for external-dns at 40, `argocd-notifications-secret` for Slack at 70, plus future app secrets at 100+). It must run before any consumer, so it sits at wave 5 — earlier than the cluster-infrastructure controllers. Its only runtime requirement is a node to schedule on, which the EKS managed node group satisfies.

## Numbering rules to keep

- Multiples of 10 for new top-level components.
- Adjacent integers (`N+1`) for configuration CRs that depend on a controller's CRDs.
- Leave the 100→app gap intact so cert-manager, OPA Gatekeeper, Kyverno, KEDA, etc. can land at 80–90 without renumbering apps.
- The `sync-wave` annotation is what ArgoCD reads; the filename prefix is for humans. Keep them aligned to avoid confusion.

## Cross-repo dependencies (debugging index)

For each wave that needs something from the infra repo, this table lists exactly what the infra side must provide. When a wave is failing, start here: confirm every infra item on the row is present and correct before debugging the platform manifest.

| Wave | Platform (this repo)                 | Infra (Terraform repo)                                                                                             |
| ---- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| 5    | External Secrets Operator (Helm)     | ESO IAM role (IRSA, `ssm:GetParameter*` + optional `kms:Decrypt`)                                                  |
| 6    | ClusterSecretStore + ExternalSecrets | SSM parameters under the agreed path prefix (e.g. `/gitops/*`)                                                     |
| 10   | Karpenter controller (Helm)          | Karpenter IAM role (IRSA), SQS interruption queue, EC2 instance profile, subnet/SG tags (`karpenter.sh/discovery`) |
| 11   | Karpenter NodePool + EC2NodeClass    | Same tags on subnets/SGs referenced in EC2NodeClass selectors                                                      |
| 20   | AWS Load Balancer Controller (Helm)  | ALBC IAM role (IRSA), VPC ID, subnet tags (`kubernetes.io/role/elb`, `kubernetes.io/role/internal-elb`)            |
| 30   | Envoy Gateway controller (Helm)      | None directly — relies on ALBC (wave 20) for LB provisioning                                                       |
| 31   | GatewayClass + Gateway               | None directly — ALB is created via wave 20's controller                                                            |
| 40   | External-DNS (Helm)                  | Cloudflare uses an API token from SSM (wave 6); if migrated to Route53, would also need an IRSA role + hosted zone |
| 70   | ArgoCD Notifications (Helm)          | Slack token in SSM (wave 6)                                                                                        |
| 90   | Argo Rollouts (Helm)                 | None                                                                                                               |
| 100+ | User applications                    | Per-app SSM parameters (wave 6), any app-specific IAM roles (RDS access, S3 buckets, etc.)                         |

Waves not listed (or with "None") have no infra-repo dependency — failures there are platform-side only.

## Monitoring

Monitoring is **handled externally by Grafana Cloud**, not by an in-cluster stack. The deferred wave-60 slot (`kube-prometheus-stack`) was intentionally skipped because:

- Grafana Cloud already ingests EKS control-plane metrics, EC2 node metrics, ALB metrics, and CloudWatch Logs via the AWS CloudWatch integration — covering the bulk of cluster-health observability without anything running in-cluster.
- Running Prometheus + Grafana + Alertmanager + exporters in-cluster would duplicate that work, consume ~2–3 GB RAM, and add CRD churn / upgrade pain (see the immutable-selector incident in the repo history).
- The principle "monitoring before apps" is still satisfied — by the external pipeline, not by an in-cluster stack.

**When to add an in-cluster collector (e.g. Grafana Alloy → remote_write to Grafana Cloud):**

- Apps start exposing `/metrics` and you want their custom metrics in Grafana.
- You need `kube-state-metrics` data (pod restarts, Deployment status, PVC usage) that CloudWatch doesn't expose.
- You want distributed tracing (Alloy ships OTel traces alongside metrics).

When that happens, the new component would land at **wave 80** as a small Alloy DaemonSet — not a re-installation of the full kube-prometheus-stack. The wave-60 slot stays reserved and unused.
