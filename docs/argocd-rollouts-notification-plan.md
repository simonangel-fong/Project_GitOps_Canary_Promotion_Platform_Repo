# Argo CD & Argo Rollouts — Notification Setup Plan

## Overview

Two separate notification engines, each watching different concerns:

|                      | ArgoCD Notifications              | Argo Rollouts Notifications           |
| -------------------- | --------------------------------- | ------------------------------------- |
| **Watches**          | `Application` CRDs                | `Rollout` & `AnalysisRun` CRDs        |
| **Config namespace** | `argocd`                          | `argo-rollouts`                       |
| **Use case**         | Sync/health of all apps & add-ons | Canary progression & analysis results |
| **Managed by**       | Terraform (ArgoCD bootstrap)      | ArgoCD (platform add-ons layer)       |

**Rule:** Use both — they are complementary, not competing.

---

## Responsibility Boundary

```
Terraform (infra bootstrap)
  └─ Deploys ArgoCD via Helm
  └─ Enables notifications controller flag
  └─ Stores Slack token in AWS SSM Parameter Store

ArgoCD (platform layer — GitOps)
  └─ Deploys Argo Rollouts via Helm Application
  └─ Manages argocd-notifications-cm (ConfigMap)
  └─ Manages argo-rollouts-notification-configmap (ConfigMap)
  └─ Manages ExternalSecret (ESO) to pull Slack token into both namespaces
```

---

## What Triggers What

### ArgoCD Notifications — covers ALL Applications (add-ons + workloads)

| Event                   | Trigger              | Channel                 |
| ----------------------- | -------------------- | ----------------------- |
| Any app sync failed     | `on-sync-failed`     | `#platform-alerts`      |
| Any app health degraded | `on-health-degraded` | `#platform-alerts`      |
| App sync succeeded      | `on-sync-succeeded`  | `#platform-deployments` |

> Add-ons (ESO, Karpenter, ALBC, ExternalDNS, Envoy) are plain ArgoCD Applications — their health is fully covered here. Do NOT put infrastructure add-ons through Argo Rollouts.

### Argo Rollouts Notifications — covers application Rollout CRDs only

| Event                        | Trigger                  | Channel                            |
| ---------------------------- | ------------------------ | ---------------------------------- |
| Canary fully promoted        | `on-rollout-completed`   | `#deployments`                     |
| Rollout aborted/degraded     | `on-rollout-degraded`    | `#deployments`, `#platform-alerts` |
| Analysis run failed          | `on-analysis-run-failed` | `#deployments`, `#platform-alerts` |
| Analysis infra/query error   | `on-analysis-run-error`  | `#platform-alerts`                 |
| Rollout paused (manual gate) | `on-rollout-paused`      | `#deployments`                     |

---

## Slack Channel Strategy

```
#platform-alerts        ← High-signal failures requiring immediate human action
                           (sync failures, health degraded, analysis errors)

#platform-deployments   ← Informational sync events for all apps and add-ons
                           (succeeded syncs, routine updates)

#deployments            ← Canary progression events for application teams
                           (step completions, promotions, paused gates)
```

---

## Implementation Steps

### Step 1 — Store Slack Token in AWS SSM Parameter Store

Before any Kubernetes config, store the credential in AWS. This keeps secrets out of Git and Terraform state.

```bash
aws ssm put-parameter \
  --name /project/env/slack/token \
  --value "xoxb-your-slack-bot-token" \
  --type SecureString \
  --region <your-region>
```

This parameter will be pulled by ESO into both namespaces (`argocd` and `argo-rollouts`).

---

### Step 2 — Enable ArgoCD Notifications Controller in Terraform

ArgoCD is deployed via Terraform + Helm. The only change needed here is enabling the notifications controller flag and setting the ArgoCD URL. **No notification content (ConfigMaps, secrets) is managed by Terraform** — that stays in Git/GitOps.

```hcl
resource "helm_release" "argocd" {
  name       = "argocd"
  namespace  = "argocd"
  chart      = "argo-cd"
  repository = "https://argoproj.github.io/argo-helm"

  values = [<<-EOT
    notifications:
      enabled: true
      argocdUrl: "https://argocd.your-domain.com"
  EOT
  ]
}
```

> Keep Terraform's role minimal: just enable the controller. All ConfigMap content is managed by ArgoCD itself in the platform layer.

---

### Step 3 — Deploy ESO ExternalSecrets for Slack Token (ArgoCD GitOps)

Since ESO is already deployed as a platform add-on, create `ExternalSecret` resources for both namespaces. These pull the SSM parameter and create the notification secrets automatically.

**For ArgoCD namespace** (`argocd-notifications-secret`):

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager # or your ClusterSecretStore name
    kind: ClusterSecretStore
  target:
    name: argocd-notifications-secret
  data:
    - secretKey: slack-token
      remoteRef:
        key: /platform/notifications/slack-token
```

**For Argo Rollouts namespace** (`argo-rollouts-notification-secret`): same pattern, target namespace `argo-rollouts`.

Both `ExternalSecret` manifests live in Git and are applied by ArgoCD.

---

### Step 4 — Configure ArgoCD Notifications ConfigMap (ArgoCD GitOps)

Create `argocd-notifications-cm` in the `argocd` namespace. This is a GitOps-managed manifest — **not Terraform**.

Key structure:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token          # references the secret key

  subscriptions: | # global — applies to ALL Applications
    - recipients:
        - slack:#platform-deployments
      triggers:
        - on-sync-succeeded
    - recipients:
        - slack:#platform-alerts
      triggers:
        - on-sync-failed
        - on-health-degraded

  template.app-sync-failed: | # include app name, namespace, phase, ArgoCD URL
    ...

  template.app-health-degraded: |
    ...

  template.app-sync-succeeded: |
    ...

  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]
  # ... other triggers
```

Place this manifest in your platform GitOps repo alongside other ArgoCD configurations.

---

### Step 5 — Deploy Argo Rollouts with Notification Config (ArgoCD GitOps)

Argo Rollouts is deployed as an ArgoCD Application via the upstream `argo-rollouts` Helm chart. The chart **unconditionally renders** `argo-rollouts-notification-configmap` (there is no flag to disable it). Therefore notification content (notifiers, templates, triggers) MUST be supplied as Helm values to that same Application — splitting it into a separate ConfigMap manifest creates two competing owners of the same resource.

Set the following in the Argo Rollouts Application's Helm values:

```yaml
notifications:
  configmap:
    create: true
  secret:
    create: false   # ESO owns argo-rollouts-notification-secret

  notifiers:
    service.slack: |
      token: $slack-token

  templates:
    template.rollout-completed: |
      message: |
        :rocket: Rollout *{{.rollout.metadata.name}}* in `{{.rollout.metadata.namespace}}` fully promoted.
      slack:
        attachments: |
          [{ "title": "{{ .rollout.metadata.name }}", "color": "#18be52", "fields": [ ... ] }]
    template.rollout-degraded: |
      ...
    template.rollout-paused: |
      ...
    template.analysis-run-failed: |
      ...
    template.analysis-run-error: |
      ...

  triggers:
    trigger.on-rollout-completed: |
      - when: rollout.status.phase == 'Healthy'
        send: [rollout-completed]
    trigger.on-rollout-degraded: |
      - when: rollout.status.phase == 'Degraded'
        send: [rollout-degraded]
    # ... other triggers
```

> Argo Rollouts' notifications engine is built into the controller — there is **no `notifications.enabled` flag**. It reads `argo-rollouts-notification-configmap` and `argo-rollouts-notification-secret` from its own namespace at runtime.

---

### Step 6 — (merged into Step 5)

In the original plan this was a separate ConfigMap Application. That approach causes a "resource is part of two applications" conflict because the chart already renders the ConfigMap. Keep this step as a placeholder for the templates/triggers content — but author it inside the Helm values block of Step 5, not as a standalone manifest.

---

### Step 7 — Annotate Rollout Resources

Subscriptions for Argo Rollouts notifications go on the **Rollout resource** (not the ArgoCD Application). Add annotations in your application Helm chart or manifest:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
  annotations:
    notifications.argoproj.io/subscribe.on-rollout-completed.slack: "#deployments"
    notifications.argoproj.io/subscribe.on-rollout-degraded.slack: "#deployments,#platform-alerts"
    notifications.argoproj.io/subscribe.on-analysis-run-failed.slack: "#deployments,#platform-alerts"
    notifications.argoproj.io/subscribe.on-rollout-paused.slack: "#deployments"
```

For ArgoCD Applications, global subscriptions in the ConfigMap (Step 4) already cover all apps — per-app annotations are only needed for overrides (e.g., routing a specific add-on to a different channel).

---

### Step 8 — Validate

```bash
# Check ArgoCD notifications controller is running
kubectl get pods -n argocd -l app.kubernetes.io/component=notifications-controller

# Check Argo Rollouts controller logs for notification events
kubectl logs -n argo-rollouts deployment/argo-rollouts | grep -i notif

# Trigger a test notification manually (ArgoCD)
kubectl patch app <app-name> -n argocd \
  --type merge -p '{"metadata":{"annotations":{"notifications.argoproj.io/subscribe.on-sync-succeeded.slack":"#platform-deployments"}}}'

# Verify ExternalSecrets are synced
kubectl get externalsecret -n argocd
kubectl get externalsecret -n argo-rollouts
```

---

## File & Repo Structure

```
platform-gitops-repo/
├── bootstrap/
│   ├── 111_external-secrets-config.yaml       # Step 3: ESO Slack ExternalSecrets (both namespaces)
│   ├── 112_argocd-notifications.yaml          # Step 4: ApplicationSet for argocd notif CM
│   └── 500_argo-rollouts.yaml                 # Step 5+6: Rollouts chart with inline notification values
└── platform/
    ├── argocd-notifications/                  # Step 4: Helm chart for argocd-notifications-cm
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   └── templates/notifications-cm.yaml
    └── external-secrets/                      # Step 3: ESO chart (renders Slack ExternalSecrets)

infra-terraform/
└── argocd.tf                                  # Helm release with notifications controller enabled (Step 2)

aws-ssm/
└── /platform/notifications/slack-token        # Step 1 (managed outside Git)
```

> The Argo Rollouts notification ConfigMap is **not** a separate chart — it is rendered by the upstream `argo-rollouts` Helm chart, driven by values in `bootstrap/500_argo-rollouts.yaml`.

---

## Summary

```
Step 1  AWS SSM          Store Slack token securely
Step 2  Terraform        Enable notifications controller in ArgoCD Helm release only
Step 3  GitOps (ESO)     ExternalSecrets pull SSM token into argocd + argo-rollouts namespaces
Step 4  GitOps           argocd-notifications-cm — global triggers for all Applications
Step 5+6 GitOps          Argo Rollouts Helm Application with notifiers/templates/triggers inlined as Helm values
                          (the chart owns the ConfigMap; no separate Application)
Step 7  App manifests    Annotate Rollout CRDs to subscribe to specific channels
Step 8  Ops              Validate controllers, secrets, and test a notification
```

Terraform touches only Step 2. Everything else is GitOps or AWS console/CLI.

---

## Implementation pitfalls (lessons learned)

- **`argo-rollouts-notification-configmap` has exactly one owner.** The upstream chart renders it unconditionally — keep notification content inline in the chart's Helm values, never in a sibling Application. A second Application targeting the same ConfigMap triggers "resource is part of two applications" warnings and an overwrite race on every sync.
- **Argo notifications secret syntax is `$secretKey`, not `${secretKey}`.** Curly braces are not recognized by either notifications engine; the literal string `${slack-token}` will be POSTed to Slack and rejected as `invalid_auth`.
- **There is no `notifications.enabled` flag for Argo Rollouts.** The notifications engine is built into the controller and always active. Argo CD does have a separate notifications controller toggled in the Helm chart (`notifications.enabled: true`) — don't conflate the two.
- **The ESO ExternalSecret must exist in the target namespace before the controller starts**, otherwise the controller fails to resolve `$slack-token` on first reconciliation. Pre-create namespaces at an early wave and let ESO sync ahead of the Helm install.
- **Never paste a real Slack token into README / docs.** GitHub Push Protection will block the push and the token must be rotated even if the push was rejected (it has already left your machine in the pack file).
