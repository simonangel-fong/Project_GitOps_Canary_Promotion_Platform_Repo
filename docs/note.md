

```sh
kubectl -n external-dns create secret generic cloudflare-api-key --from-literal=cf_api_token=<cf_key>
```