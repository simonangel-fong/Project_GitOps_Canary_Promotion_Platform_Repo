# Runbook - ArgoCD App-of-Apps

[Back](../README.md)

- [Runbook - ArgoCD App-of-Apps](#runbook---argocd-app-of-apps)
  - [Access ArgoCD UI](#access-argocd-ui)
  - [Issue: Delete Get Stuck](#issue-delete-get-stuck)
  - [Prometheus Metrics](#prometheus-metrics)

---

## Access ArgoCD UI

```sh
# update kube config
aws eks update-kubeconfig --region ca-central-1 --name gitops-demo-dev

# forward
kubectl port-forward svc/argocd-server 8000:80 -n argocd
kubectl port-forward svc/argo-rollouts-dashboard 3100:3100 -n argo-rollouts

# get initial-admin-secret
k get secret argocd-initial-admin-secret -n argocd -o yaml
# decode
echo "" | base64 -d

# sync
argocd app sync argocd/00-app-of-apps

```

## Issue: Delete Get Stuck

- Common causes: `metadata.finalizers`
- Solution: patch by removing finalizers

```sh
kubectl delete application <root-app-name> -n argocd

# EnvoyProxy
kubectl patch EnvoyProxy eg-nlb -p '{"metadata":{"finalizers":[]}}' --type=merge -n envoy-gateway-system
# GatewayClass
kubectl patch GatewayClass eg -p '{"metadata":{"finalizers":[]}}' --type=merge -n envoy-gateway-system
# EC2NodeClass
kubectl patch EC2NodeClass general-node-class -p '{"metadata":{"finalizers":[]}}' --type=merge
# namespace
kubectl patch namespace external-dns -p '{"metadata":{"finalizers":[]}}' --type=merge
# app
kubectl patch app platform-600-kube-prometheus-stack -p '{"metadata":{"finalizers":[]}}' --type=merge -n argocd
```

## Prometheus Metrics

- Confirm before Argo Rollouts AnalysisTemplate

```sh
# forward
kubectl port-forward -n monitoring svc/kps-prometheus 9090:9090
# browser: http://localhost:9090/query?g0.expr=kube_pod_container_status_restarts_total%7Bnamespace%3D%22backend%22%7D

# or using command
kubectl run -n monitoring promcheck --rm -it --image=curlimages/curl --restart=Never -- curl -s "http://kps-prometheus.monitoring.svc.cluster.local:9090/api/v1/query?query=kube_pod_container_status_restarts_total%7Bnamespace%3D%22backend%22%7D"
# {"status":"success","data":{"resultType":"vector","result":[{"metric":{"__name__":"kube_pod_container_status_restarts_total","container":"gitops-backend","endpoint":"http","instance":"10.0.11.15:8080","job":"kube-state-metrics","namespace":"backend","pod":"gitops-backend-7454bb5d99-tknhq","service":"kube-prometheus-stack-kube-state-metrics","uid":"b30d3ca4-d7b5-417a-b2bb-a961bc31a80d"},"value":[1779773364.635,"0"]},{"metric":{"__name__":"kube_pod_container_status_restarts_total","container":"gitops-backend","endpoint":"http","instance":"10.0.11.15:8080","job":"kube-state-metrics","namespace":"backend","pod":"gitops-backend-7454bb5d99-cq6fz","service":"kube-prometheus-stack-kube-state-metrics","uid":"c58a0a47-b9b8-4d3f-8690-32c80a8023fc"},"value":[1779773364.635,"0"]},{"metric":{"__name__":"kube_pod_container_status_restarts_total","container":"gitops-backend","endpoint":"http","instance":"10.0.11.15:8080","job":"kube-state-metrics","namespace":"backend","pod":"gitops-backend-7454bb5d99-qdxm9","service":"kube-prometheus-stack-kube-state-metrics","uid":"e59e0391-9a18-4aab-a2be-563fb4d077a7"},"value":[1779773364.635,"0"]},{"metric":{"__name__":"kube_pod_container_status_restarts_total","container":"gitops-backend","endpoint":"http","instance":"10.0.11.15:8080","job":"kube-state-metrics","namespace":"backend","pod":"gitops-backend-7454bb5d99-p6lpg","service":"kube-prometheus-stack-kube-state-metrics","uid":"a4ccf6ce-f6e5-4b1e-b6b8-baa833144dd0"},"value":[1779773364.635,"0"]}]}}pod "promcheck" deleted from monitoring namespace
```
