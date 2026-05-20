
```sh
kubectl port-forward svc/gitops-demo-backend 8080:8080 -n backend

kubectl port-forward svc/gitops-demo-frontend 8800:80 -n frontend




curl -H "host:gitops-dev.arguswatcher.net" gitops-demo-dev-76d293eb919aace1.elb.ca-central-1.amazonaws.com
```