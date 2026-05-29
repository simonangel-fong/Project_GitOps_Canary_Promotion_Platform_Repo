# Canary Deployment Design — Dev → Stage

## Purpose

Demonstrate Argo Rollouts as a progressive-delivery filter across two environments, using one image and three failure scenarios. Show _which layer_ catches _which bug class_, and where canary's limits are.

This document is the design contract for the build. A diagram of the full vision (including the unbuilt prod tier, soak gate, and alerting) is referenced at the end and lives separately.

## The matrix

| Scenario                         | Env   | Bug                      | Canary config                                        | Catching layer                               | Outcome                                                                      |
| -------------------------------- | ----- | ------------------------ | ---------------------------------------------------- | -------------------------------------------- | ---------------------------------------------------------------------------- |
| 1. Baseline                      | dev   | none                     | 25/50/100, 60s pauses                                | n/a                                          | Promotes cleanly                                                             |
| 2. RDS connection failure        | dev   | bug 1                    | 25/50/100, 60s pauses                                | `readinessProbe` + `progressDeadlineSeconds` | Pod never Ready → rollout aborts during 25% step                             |
| 3a. Subtle OOM — escapes dev     | dev   | bug 2                    | 25/50/100, 60s pauses                                | none — bug outlasts canary window            | Rollout completes; pods OOM post-promotion; manual recovery via `git revert` |
| 3b. Subtle OOM — caught in stage | stage | bug 2 (same image as 3a) | 25/50/100, 10min pauses + AnalysisTemplate on memory | AnalysisTemplate (Prometheus memory query)   | Memory climbs during 25% step → analysis fails → auto-rollback               |

One image. Two overlays. Three runs. Each scenario exercises a distinct catching mechanism.

## Thesis

Canary is a filter, not a guarantee. Whether a bug is caught depends on whether the canary window outlasts the bug's incubation period. Scenarios 3a and 3b are the same bug, same image — only the gate length differs, and the outcome flips. That is the point.

## Bug definitions

### Bug 1 — RDS connection failure

- Trigger: env vars `PGDB_ENABLE=true` and `PGDB_URL` pointed at an unreachable host (e.g. `jdbc:postgresql://broken-host:5432/demo_db`).
- Symptom: Spring Boot attempts to connect on startup / health-check time, `/api/healthz` returns non-200 (the endpoint includes a DB check).
- Manifests as: readinessProbe fails on every new pod.
- Catching layer: Kubernetes readiness + Argo Rollouts `progressDeadlineSeconds`. AnalysisTemplate is _not needed_ — canary pods never become Ready, so the rollout cannot progress.

### Bug 2 — Subtle OOM

- Trigger: env vars `OOM_ENABLE=true` and `OOM_TIME=N` on the Spring Boot app.
- Behavior: app starts and serves traffic normally. After N minutes, allocates past the 256Mi container limit → OOMKilled → restarts → OOMs again → CrashLoopBackOff.
- Catching layer depends on canary window length:
  - **Window < OOM_TIME** (dev, 60s pauses) → rollout completes before OOM fires. Bug escapes.
  - **Window > OOM_TIME** (stage, 10min pauses) → OOM fires while canary pod is serving 25% of traffic. AnalysisTemplate sees memory spike or pod restart and aborts.

## Argo Rollouts configuration

### Dev — fast canary

```yaml
strategy:
  canary:
    steps:
      - setWeight: 25
      - pause: { duration: 60s }
      - setWeight: 50
      - pause: { duration: 60s }
      - setWeight: 100
progressDeadlineSeconds: 120
progressDeadlineAbort: true
```

- Total rollout time: ~3 min.
- `progressDeadlineSeconds: 120` aborts if a canary pod isn't Ready within 2 min — this is what catches bug 1.
- No AnalysisTemplate. Dev is fast and accepts that delayed bugs may slip through. That is the design trade.

### Stage — long canary + analysis

```yaml
strategy:
  canary:
    steps:
      - setWeight: 25
      - pause: { duration: 10m }
      - analysis:
          templates:
            - templateName: backend-memory-health
      - setWeight: 50
      - pause: { duration: 10m }
      - analysis:
          templates:
            - templateName: backend-memory-health
      - setWeight: 100
progressDeadlineSeconds: 900
progressDeadlineAbort: true
```

- Total rollout time: ~20 min if it completes.
- Bug 2's OOM (configured to fire at ~3 min) lands well inside the 10-min canary window.
- AnalysisTemplate runs after each pause and aborts on failure.

### AnalysisTemplate (stage only)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: backend-memory-health
  namespace: backend
spec:
  metrics:
    - name: pod-restart-count
      interval: 60s
      count: 3
      successCondition: result[0] == 0
      failureLimit: 1
      provider:
        prometheus:
          address: http://kps-prometheus.monitoring.svc.cluster.local:9090
          query: |
            sum(increase(
              kube_pod_container_status_restarts_total{
                namespace="backend",
                pod=~"gitops-backend-.*"
              }[2m]
            ))
```

- Single metric: container restart count in the last 2 minutes.
- An OOMKill produces a restart. With `failureLimit: 1`, the analysis tolerates **up to 1 failed measurement** across the 3-sample window before aborting — i.e. 2 of 3 measurements must report a restart for the analysis itself to fail. (Argo Rollouts semantics: `failureLimit` is the count of allowed failures, not the threshold that triggers abort.)
- Memory-utilization query was considered but rejected: noisier, requires tuning a threshold, and the restart signal is unambiguous.

#### Observed during dev rehearsal

First analysis-step result: `✔ 2, ✖ 1` — passed (one failure tolerated, advanced to 50%). Second analysis-step result: `✖ 2` on the first two of three measurements — aborted as soon as the second failure tipped past `failureLimit`. Both windows were inside tolerance individually for the first run, but a sustained OOM loop produces consistent failures across measurement windows, so the second analysis catches what the first nearly missed. The two-analysis structure (one per pause) is what makes this robust, not the per-analysis tuning.

Trade-off note: `failureLimit: 0` would catch on the first failed measurement (single-shot abort, no tolerance) at the cost of false-positive risk from a single transient scrape blip. Current setting prioritizes robustness against scrape jitter over fastest-possible detection.

## Rehearsal in dev (before stage)

Scenario 3b's mechanism (long pauses + AnalysisTemplate) is rehearsed against the dev cluster before running on stage. This is a deliberate departure from the design's two-environment thesis — done once, for video capture — so the demo can be filmed against a single cluster without provisioning stage. The thesis ("dev short canary misses, stage long canary catches") still describes the intended production architecture.

To rehearse, the dev overlay temporarily mirrors stage config: long pauses, analysis steps, OOM env vars on. Two JSON patches in [apps/backend/overlays/dev/kustomization.yaml](apps/backend/overlays/dev/kustomization.yaml) do the work:

```yaml
# Bug toggle — flip OOM on (same as stage will do).
- op: replace
  path: /spec/template/spec/containers/0/env/4/value
  value: "true" # OOM_ENABLE
- op: replace
  path: /spec/template/spec/containers/0/env/5/value
  value: "3"    # OOM_TIME, in minutes

# Strategy override — replace dev's short canary with stage's long canary + analysis.
- op: replace
  path: /spec/progressDeadlineSeconds
  value: 900
- op: replace
  path: /spec/strategy/canary/steps
  value:
    - setWeight: 25
    - pause: { duration: 10m }
    - analysis:
        templates:
          - templateName: backend-memory-health
    - setWeight: 50
    - pause: { duration: 10m }
    - analysis:
        templates:
          - templateName: backend-memory-health
    - setWeight: 100
```

After capture, both patches revert: OOM env vars back to `"false"` / `"1"`, the strategy/deadline patches removed entirely (so dev inherits the base's short canary). The AnalysisTemplate in [apps/backend/base/70_analysis_template.yaml](apps/backend/base/70_analysis_template.yaml) stays — it's inert when no rollout references it.

## Where the env vars live

All bug-control env vars live in [apps/backend/base/20_rollout.yaml](apps/backend/base/20_rollout.yaml) with safe defaults:

```yaml
env:
  - name: PGDB_ENABLE
    value: "false" # DB disabled by default; bug 1 demo flips to "true"
  - name: PGDB_URL
    value: "jdbc:postgresql://postgres:5432/demo_db" # valid by default
  - name: OOM_ENABLE
    value: "false"
  - name: OOM_TIME
    value: "1"
```

Demo overlays patch these on via Kustomize JSON patches, matching the existing HPA patch pattern in [apps/backend/overlays/dev/kustomization.yaml](apps/backend/overlays/dev/kustomization.yaml). Prod overlay (when added) inherits the safe defaults — no risk of demo-mode leaking.

## Rollback paths (three of them, deliberate)

| Path                | Command                                                      | When to use                                                                                                                                                               |
| ------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| In-flight abort     | `kubectl argo rollouts abort gitops-backend -n backend` | Rollout is paused or progressing and you want to stop it now. Returns traffic to stable, leaves new ReplicaSet at 0.                                                      |
| Post-promotion undo | `kubectl argo rollouts undo gitops-backend -n backend`  | Rollout has completed and you need to back out. **Drifts from Git** — ArgoCD will sync the bad version back unless self-heal is off. Emergency-only.                      |
| GitOps-canonical    | `git revert <image-bump-commit>` then push                   | Rollout has completed and the bug is confirmed. Restores desired state in Git, ArgoCD syncs the previous image. Auditable, reversible. **This is the production answer.** |

Scenario 3a's recovery uses path 3 (git revert). Path 2 is shown as the imperative escape hatch but called out as drift-inducing.

## What this design deliberately does not do

- **No prod tier.** Two clusters is enough to make the dev-vs-stage point. Prod would be a third overlay copying stage with manual promotion gates. Designed, not built.
- **No soak gating.** Soak testing — holding at 100% for hours/days post-promotion, watching SLOs, gating the next environment on the result — is a separate apparatus. Belongs between stage and prod. Designed, not built.
- **No automated rollback from alerts.** Alertmanager → auto-rollback is a known-hard problem (flapping alerts → flapping rollbacks). Alerts in this design notify humans; humans run `git revert`. This is the production-standard pattern.
- **No multi-metric AnalysisTemplate.** A real production setup would gate on success rate, latency p99, and error budget burn in addition to restart count. Single-metric analysis is sufficient to demonstrate the mechanism; tuning a multi-metric template is a depth exercise that doesn't add to the story.
- **No dev-side AnalysisTemplate.** Dev's job is fast feedback. Adding analysis to dev would either slow it down or be set so loose it'd catch nothing. The right answer is "dev accepts escape risk; stage catches what dev misses." That trade is the design.

## Build plan (one week)

| Day | Task                                                                                                                                                                                                                                                                                 |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | Confirm preconditions: stage cluster registered with ArgoCD; Prometheus reachable from stage; Spring Boot OOM reproduces reliably with `OOM_ENABLE=true OOM_TIME=3` at 256Mi limit; stage overlay syncs today.                                                                   |
| 2   | Confirm `PGDB_ENABLE`, `PGDB_URL`, `OOM_ENABLE`, `OOM_TIME` env vars exist in [apps/backend/base/20_rollout.yaml](apps/backend/base/20_rollout.yaml) with safe defaults. Patch dev and stage overlays to enable bug modes for the demo. Add `progressDeadlineSeconds` and `progressDeadlineAbort` per env. |
| 3   | Write AnalysisTemplate. Wire into stage rollout steps. Verify Prometheus service DNS from inside the stage cluster.                                                                                                                                                                  |
| 4   | Dry-run all three scenarios end-to-end on the live clusters. Tune timings if OOM doesn't land inside the canary window. Capture asciinema/screenshots of `kubectl argo rollouts get rollout -w` for each run.                                                                        |
| 5   | Draft README. Build the architecture diagram (built vs designed, solid vs dashed).                                                                                                                                                                                                   |
| 6   | Record + edit the 2-min video.                                                                                                                                                                                                                                                       |
| 7   | Polish README. Write the "what this deliberately does not do" section. Final pass. Push.                                                                                                                                                                                             |

Day 1 is non-negotiable. If OOM doesn't reproduce reliably or Prometheus isn't reachable from stage, the rest of the plan is built on sand.

## Preconditions (day-1 checklist)

- [ ] Stage cluster registered with ArgoCD — `kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=cluster` shows it.
- [ ] Backend ApplicationSet ([bootstrap/900_app_backend.yaml](bootstrap/900_app_backend.yaml)) generates a stage Application — `kubectl get applications -n argocd` shows `app-900-backend` for stage.
- [ ] Prometheus reachable from inside the stage cluster — `kubectl run -n backend --rm -it --image=curlimages/curl curl-test -- curl http://prometheus-server.monitoring.svc.cluster.local:9090/-/ready` returns 200.
- [ ] `kube_pod_container_status_restarts_total` metric exists — query it in the Prometheus UI before relying on it in the AnalysisTemplate.
- [ ] Spring Boot OOM fires within window — local docker run with `OOM_ENABLE=true OOM_TIME=3 -m 256m`, confirm OOMKill within 4-5 min.
- [ ] HPA on dev allows enough replicas — `maxReplicas: 2` may round 25% canary weight down to 0 pods. Bump to 4 for the demo if needed.

## Open design questions to resolve during build

1. ~~**Bug 1 trigger mechanism.**~~ Resolved: use `PGDB_ENABLE=true` + unreachable `PGDB_URL`. `/api/healthz` includes a DB check, so the readiness probe fails when the DB is unreachable.
2. **Should scenario 3a end with `git revert` on camera, or just describe it?** Recording the revert adds 30s to the video but completes the story. Decide during day-6 edit.
3. **AnalysisTemplate placement.** Currently in [apps/backend/base/](apps/backend/base/) and applied to all overlays via a stage-only patch, or in [apps/backend/overlays/stage/](apps/backend/overlays/stage/) directly. The latter is simpler; the former is more "production" (template lives with the app, environments toggle it on). Resolve on day 3.

## Reference

- Argo Rollouts canary spec: https://argoproj.github.io/argo-rollouts/features/canary/
- AnalysisTemplate spec: https://argoproj.github.io/argo-rollouts/features/analysis/
- `progressDeadlineSeconds` semantics: https://argoproj.github.io/argo-rollouts/features/rollout-spec/#progressdeadlineseconds

---


case 3a

```sh
# revert
git revert 43d66eb --no-edit
git push
```
