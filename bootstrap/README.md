
```sh
aws eks update-kubeconfig --region ca-central-1 --name gitops-demo-dev

kubectl port-forward svc/argocd-server 8000:80 -n argocd

kubectl port-forward svc/argo-rollouts-dashboard 3100:3100 -n argo-rollouts

k get secret argocd-initial-admin-secret -n argocd -o yaml

echo "" | base64 -d

```

- envoy

```sh
argocd repo add docker.io/envoyproxy --type helm --name envoyproxy --enable-oci


helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.8.0 -n envoy-gateway-system --create-namespace

kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available

```