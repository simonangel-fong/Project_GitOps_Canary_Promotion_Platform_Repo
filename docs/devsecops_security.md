# Local Lint & Security Scan — Developer Guide

[Back](../README.md)

---

## Prerequisites

Install:

| Tool        | Version | Install                                               |
| ----------- | ------- | ----------------------------------------------------- |
| kubeconform | v0.6.7  | https://github.com/yannh/kubeconform/releases         |
| kustomize   | v5.4.3  | `choco install kustomize` / `brew install kustomize`  |
| helm        | v3.15.3 | `choco install kubernetes-helm` / `brew install helm` |
| python      | 3.12    | https://www.python.org/downloads/                     |
| checkov     | 3.2.334 | `pip install checkov==3.2.334`                        |

Verify:

```bash
# install kubeconform
winget install kubeconform
kubeconform -v
kustomize version
helm version --short
checkov --version
```

---

## Lint Check

Renders manifests, then validates against the Kubernetes schema. CRD schemas are intentionally skipped (`-ignore-missing-schemas`) — Argo CRDs, External Secrets, Karpenter, etc.

### Lint a single app overlay

```bash
kustomize build apps/backend/overlays/dev | kubeconform -verbose -strict -ignore-missing-schemas -summary -
# stdin - gitops-backend Rollout skipped
# stdin - ServiceAccount gitops-backend is valid
# stdin - Service gitops-backend is valid
# stdin - HorizontalPodAutoscaler gitops-backend is valid
# stdin - NetworkPolicy allow-dns-egress is valid
# stdin - NetworkPolicy default-deny is valid
# stdin - NetworkPolicy allow-ingress-from-frontend is valid
# Summary: 7 resources found parsing stdin - Valid: 6, Invalid: 0, Errors: 0, Skipped: 1
```

### Lint all app overlays

Bash:

```bash
for d in apps/*/overlays/*/; do
  echo "==> $d"
  kustomize build "$d" | kubeconform -strict -ignore-missing-schemas -summary -
done
```

### Lint a single platform chart

```bash
helm lint platform/envoy
helm template platform/envoy | kubeconform -verbose -strict -ignore-missing-schemas -summary -
```

### Lint all platform charts

Bash:

```bash
for c in platform/*/; do
  [ -f "${c}Chart.yaml" ] || continue
  echo "==> $c"
  helm lint "$c"
  helm template "$c" | kubeconform -verbose -strict -ignore-missing-schemas -summary -
done
```

### 2.5 Lint bootstrap manifests

```bash
kubeconform -verbose -strict -ignore-missing-schemas -summary bootstrap/*.yaml
```

---

## Security scan

Checkov runs against all three frameworks at once. Skip-list lives in [.checkov.yaml](../.checkov.yaml).

### 3.1 Scan everything (what CI runs)

```bash
# windows
docker run --rm -v "${PWD}:/code" bridgecrew/checkov `
  --directory .`
  --framework kubernetes  `
  --config-file .checkov.yaml   `
  -output cli   `
  --compact

checkov --directory . --config-file .checkov.yaml --output cli --compact
```

### Scan a single path

```bash
checkov --directory apps/backend --config-file .checkov.yaml --output cli --compact
```

### Get the check ID for a finding (to add to skip-list)

The check ID appears in the CLI output as `Check: CKV_K8S_xx`. Add it to `.checkov.yaml` under `skip-check:` with a one-line reason.

```yaml
skip-check:
  - CKV_K8S_43 # image tag pinned via Kustomize images[]; check sees placeholder
```

### List all available checks

```bash
checkov --list
```

---

## Before opening a PR — quick checklist

```bash
# 1. Lint everything changed
kustomize build apps/<svc>/overlays/<env> | kubeconform -strict -ignore-missing-schemas -summary -
helm template platform/<chart> | kubeconform -strict -ignore-missing-schemas -summary -

# 2. Scan
checkov --directory . --config-file .checkov.yaml --output cli --compact
```

If both pass locally, CI will pass.

---

## 5. Troubleshooting

| Symptom                                                        | Cause / Fix                                                                                         |
| -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `could not find schema for <CRD>`                              | Expected — CRDs are skipped via `-ignore-missing-schemas`. Not an error.                            |
| `kustomize build` errors on `patches:`                         | Your overlay references a resource that isn't in `base/`. Run `kustomize build` and read the trace. |
| `helm lint` warns `icon is recommended`                        | Cosmetic. Add `icon:` to `Chart.yaml` or ignore.                                                    |
| Checkov flags a finding you want to ignore globally            | Add the `CKV_*` ID to `.checkov.yaml` `skip-check:` with a one-line reason.                         |
| Checkov flags a finding you want to ignore on **one resource** | Add `# checkov:skip=CKV_K8S_43:reason` as a YAML comment on the resource.                           |
| Different versions of a tool produce different findings        | Match CI versions exactly (see §1 table).                                                           |
