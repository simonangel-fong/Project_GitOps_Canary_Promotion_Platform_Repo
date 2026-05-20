
```sh
kubectl port-forward svc/argocd-server 8000:80 -n argocd

kubectl port-forward svc/argo-rollouts-dashboard 3100:3100 -n argo-rollouts
```