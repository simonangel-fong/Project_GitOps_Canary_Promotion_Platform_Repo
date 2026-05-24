# CI/CD Plan — Project_GitOps_Platform_Repo

GitOps manifest repo. Argo CD reconciles `dev`, `stage`, `prod` branches onto EKS. GitHub Actions handles lint, scan, and promotion-PR orchestration. Deployment safety is enforced in-cluster by Argo Rollouts + k6 AnalysisRun.

---

## 1. Goals & Non-Goals

**Goals**

- GitOps flow: branch state = cluster state, reconciled by Argo CD.
- Lint and security-scan manifests on every change.
- Automated promotion `dev → stage` (PR auto-merge); human-gated promotion `stage → prod` (PR review).
- In-cluster deploy verification via Argo Rollouts AnalysisRun (k6).
- Deploy event notifications via Argo Notifications → GitHub `repository_dispatch` + Slack.

**Non-Goals**

- Application repo pipeline.
- Infrastructure repo pipeline (EKS, Argo CD install, IAM, Argo Notifications service config).
- External smoke/load tests (deferred — see §8).

---

## 2. Branching & Environment Model

| Environment | Branch  | GH Actions Environment | Endpoint                      |
| ----------- | ------- | ---------------------- | ----------------------------- |
| dev         | `dev`   | `dev`                  | gitops-dev.arguswatcher.net   |
| stage       | `stage` | `stage`                | gitops-stage.arguswatcher.net |
| prod        | `prod`  | `prod`                 | gitops-prod.arguswatcher.net  |

No `main` branch. Prod approval is enforced by **branch protection on `prod`** (required reviewers on the `stage → prod` PR). Merge to `prod` = deploy via Argo CD.

Per-env values come from GitHub Environment secrets/variables at runtime. Nothing per-env is committed.

**Repo layout:**

```
.github/      GitHub Actions workflows + composite actions
apps/         Application manifests (Kustomize: base + overlays)
platform/     Platform component charts (Helm)
bootstrap/    App-of-apps definitions (raw YAML)
docs/         Documentation
```

---

## 3. Pipeline Inventory

| #   | Workflow file       | Trigger                                                          | Purpose                                              |
| --- | ------------------- | ---------------------------------------------------------------- | ---------------------------------------------------- |
| 1   | `10-ci-check.yml`   | push `feature-*`/`hotfix-*`; PR → `dev`/`stage`/`prod`           | Lint + scan. Bot-actor PRs skip scan (tag-only).     |
| 2   | `20-cd-dev.yml`     | `repository_dispatch: argocd-deployed-dev`; `workflow_dispatch`  | Open auto-merge promotion PR `dev → stage`.          |
| 3   | `21-cd-stage.yml`   | `repository_dispatch: argocd-deployed-stage`; `workflow_dispatch`| Open human-review promotion PR `stage → prod`.       |
| 4   | `30-cd-prod.yml`    | `repository_dispatch: argocd-deployed-prod`; `workflow_dispatch` | Post release announcement to Slack.                  |

Composite actions in [.github/actions/](../.github/actions/) are shared.

---

## 4. Composite Actions

| Action          | Tool(s)                                              | Notes                                                              |
| --------------- | ---------------------------------------------------- | ------------------------------------------------------------------ |
| `lint-check`    | `kubeconform`, `kustomize build`, `helm lint/template` | Render `apps/` (kustomize) and `platform/` (helm), then validate. |
| `security-scan` | `checkov --framework kubernetes,helm,kustomize`      | Single scanner across all three formats.                           |
| `notify-slack`  | Slack bot token (`SLACK_BOT_TOKEN`)                  | `if: always()` final job in every workflow.                        |

Image vulnerability scanning lives in the application repo (scans the built image before pushing). Smoke/load testing is delegated to Argo Rollouts AnalysisRun in-cluster (§6).

---

## 5. Workflow Specs

### 5.1 `10-ci-check.yml` — CI

- **Triggers**
  - `push`: `feature-*`, `hotfix-*` (global path filter).
  - `pull_request`: base `dev`, `stage`, `prod` (global path filter).
  - `workflow_dispatch`.
- **Path filter (global)**: `apps/**`, `platform/**`, `bootstrap/**`, `.github/**`.
- **Concurrency**: `group: ci-${{ github.ref }}`, `cancel-in-progress: true`.
- **Permissions**: `contents: read`, `pull-requests: write`, `security-events: write`, `id-token: none`.
- **Jobs**
  1. `lint-check` (parallel)
  2. `security-scan` (parallel) — **skipped if `github.actor` is the image-tag bot** (tag-only PRs need no manifest scan).
  3. `notify-slack` — `if: always()`, `needs: [lint-check, security-scan]`.

---

### 5.2 `20-cd-dev.yml` — promote dev → stage

- **Triggers**
  - `repository_dispatch`: `argocd-deployed-dev` (fired by Argo Notifications when the `apps-dev` app-of-apps reports `Healthy + Synced`).
  - `workflow_dispatch`.
- **Concurrency**: `group: cd-${{ github.workflow }}`, `cancel-in-progress: false`.
- **Permissions**: `contents: write`, `pull-requests: write`.
- **Environment**: `dev`.
- **Jobs**
  1. `open-promotion-pr` — open PR `dev → stage` from the SHA in `client_payload.revision`. Enable auto-merge. Branch protection on `stage` requires `10-ci-check.yml` green before merge.
  2. `notify-slack` — `if: always()`.

---

### 5.3 `21-cd-stage.yml` — promote stage → prod

- **Triggers**
  - `repository_dispatch`: `argocd-deployed-stage`.
  - `workflow_dispatch`.
- **Concurrency**: `group: cd-${{ github.workflow }}`, `cancel-in-progress: false`.
- **Permissions**: `contents: write`, `pull-requests: write`.
- **Environment**: `stage`.
- **Jobs**
  1. `open-promotion-pr` — open PR `stage → prod` from `client_payload.revision`. **Auto-merge disabled**; CODEOWNERS + branch protection on `prod` require human reviewer approval.
  2. `notify-slack` — `if: always()`.

---

### 5.4 `30-cd-prod.yml` — production release announcement

- **Triggers**
  - `repository_dispatch`: `argocd-deployed-prod`.
  - `workflow_dispatch`.
- **Concurrency**: `group: cd-prod`, `cancel-in-progress: false`.
- **Permissions**: `contents: read`.
- **Environment**: `prod` (GH Environment; protection rule = required reviewers, dismiss stale approvals — also enforced upstream on the `prod` branch).
- **Jobs**
  1. `notify-slack` — release announcement (env, app, revision, Argo URL).

Note: The prod approval gate is on the `stage → prod` PR review (Option A). Merge → Argo CD syncs → this workflow only announces.

---

## 6. Deployment Verification (in-cluster)

**Primary gate**: Argo Rollouts `AnalysisTemplate` runs an in-cluster k6 Job against the canary ClusterIP service during rollout. Job exit code 0 = pass → rollout promotes. Non-zero = fail → Rollouts auto-aborts and reverts to the previous ReplicaSet.

- AnalysisTemplate CRs and k6 scripts (mounted as ConfigMaps) live in this repo, per-app under `apps/<svc>/base/` and overridden per-env in `apps/<svc>/overlays/<env>/`.
- No GitHub Actions deploy gating — by the time `repository_dispatch` fires, the rollout has already passed in-cluster verification.

---

## 7. Secrets, Variables, OIDC

This repo's GitHub Actions only needs:

| Name                | Type        | Scope            | Used by         |
| ------------------- | ----------- | ---------------- | --------------- |
| `SLACK_BOT_TOKEN`   | Secret      | Repo or env      | `notify-slack`  |
| `GITHUB_TOKEN`      | Built-in    | Per-workflow     | Promotion PRs   |

**No AWS OIDC**: this repo does not touch AWS. Argo CD (in-cluster, configured by the infra repo) performs all cluster operations.

**Cross-repo writes**: the application repo opens image-tag PRs into this repo using its own GitHub App token (owned by the app repo).

**Argo Notifications split (industry "Platform as a Product" pattern):**

| Component                            | Owner            | Change frequency |
| ------------------------------------ | ---------------- | ---------------- |
| Argo Notifications controller        | Infra repo       | Rare             |
| `argocd-notifications-secret` (Slack bot token, GitHub App key) | Infra repo | Rare |
| `service.*` config in `argocd-notifications-cm`                  | Infra repo | Rare |
| `template.*` + `trigger.*` config in `argocd-notifications-cm`   | This repo  | Frequent |
| Per-Application `subscriptions` annotations                      | This repo  | Frequent |

The two `argocd-notifications-cm` halves are merged at apply time (Kustomize `configMapGenerator` with `behavior: merge`, or equivalent).

**Argo Notifications → GitHub contract:**

- Triggers fire from the **per-env app-of-apps parent Application** (e.g., `apps-dev`), not individual children → one event per successful env rollout.
- Event names: `argocd-deployed-{dev|stage|prod}` (one type per env).
- `client_payload`:
  ```json
  {
    "environment": "dev",
    "revision": "<git-sha>",
    "argocd_app_url": "https://argocd.<domain>/applications/apps-dev"
  }
  ```
- Failure triggers (`on-sync-failed`, `on-health-degraded`) send Slack alerts only; no GitHub event, no auto-revert.

---

## 8. Guardrails

**Branch protection (`dev`, `stage`, `prod`):**

- Required PR (no direct push).
- Required status checks: `lint-check`, `security-scan`.
- Linear history (squash or rebase merge only).
- `prod` only: required reviewer approvals (CODEOWNERS); dismiss stale approvals on new commits.

**CODEOWNERS** (industry "Platform as a Product" model):

```
apps/backend/    @app-backend-team
apps/frontend/   @app-frontend-team
platform/        @platform-team
bootstrap/       @platform-team
.github/         @platform-team
docs/            @platform-team
```

App teams own their `apps/<svc>/` directory (CKAD-level K8s skills assumed: Deployment, Service, ConfigMap, overlays). Platform team owns shared bases, charts, Argo Rollouts strategy templates, and the bootstrap layer. Cross-cutting safety (privileged pods, hostNetwork, missing limits) is enforced by admission policies in the platform layer, not by review vigilance.

**Bot identity**: image-tag PRs are opened by a GitHub App (`*-bot[bot]` actor). `security-scan` is skipped when `github.actor` matches the bot pattern (tag-only changes don't alter K8s posture).

**Concurrency**: CD workflows use `cancel-in-progress: false` — never cancel a half-applied state. Occasional duplicate promotion PRs are acceptable.

---

## 9. Open Items / Follow-Ups

- **External smoke + load tests** in GitHub Actions (k6 against the public endpoint) as a complementary post-rollout check.
- **Per-job path filters** (using `dorny/paths-filter`) to lint only affected `apps/<svc>/` or `platform/<chart>/` subtrees.
- **Drift detection** is handled by Argo CD itself (out-of-sync alerts via Notifications) — no separate workflow planned.
- **`argocd-notifications-cm` merge mechanism** to be confirmed with the infra repo (Kustomize merge vs Helm extras vs ApplicationSet generators).
- **Admission policies** (Kyverno or OPA Gatekeeper) for cluster-wide safety guarantees — planned in the platform layer, out of scope here.

---

## 10. Diagram

```
=========================  CI  =========================

  push feature-*/hotfix-*           PR → dev/stage/prod
            │                                │
            └──────────────┬─────────────────┘
                           ▼
                   [10-ci-check]
                  lint + scan (bot skips scan)
                           │
                           ▼
                       merge


=========================  CD  =========================

  app-repo image bump  ─PR──▶  dev branch  ◀── feature PR merge
                                   │
                                   ▼
                          ┌─────────────────┐
                          │   Argo CD sync  │
                          │  apps-dev (AoA) │
                          └─────────────────┘
                                   │
                                   ▼
                       ┌────────────────────────┐
                       │ Argo Rollouts canary   │
                       │ + k6 AnalysisRun       │  ◀── in-cluster gate
                       └────────────────────────┘
                                   │
                  Healthy + Synced │ (fail → auto-rollback)
                                   ▼
                       Argo Notifications
                       ├──▶ Slack #deploys
                       └──▶ repository_dispatch
                                   │
                                   ▼
                          [20-cd-dev]
                          open PR dev → stage
                          (auto-merge on green)
                                   │
                                   ▼
                              stage branch
                                   │
                                   ▼  (Argo sync → Rollouts → AnalysisRun → notify)
                          [21-cd-stage]
                          open PR stage → prod
                          (human review required)
                                   │
                                   ▼
                              [approval gate]
                          CODEOWNERS + branch protection
                                   │
                                   ▼
                              prod branch
                                   │
                                   ▼  (Argo sync → Rollouts → AnalysisRun → notify)
                          [30-cd-prod]
                          Slack release announcement
```
