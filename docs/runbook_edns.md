# Runbook - Debugging External DNS

[Back](../README.md)

- [Runbook - Debugging External DNS](#runbook---debugging-external-dns)
  - [No `Cloudflare` DNS Records Update](#no-cloudflare-dns-records-update)
    - [Issue](#issue)
    - [Debugging](#debugging)
    - [Root cause](#root-cause)
    - [Fix](#fix)
    - [Confirm](#confirm)
  - [HTTPS returns "unable to get local issuer certificate"](#https-returns-unable-to-get-local-issuer-certificate)
    - [Issue](#issue-1)
    - [Debugging](#debugging-1)
    - [Root cause](#root-cause-1)
    - [Fix](#fix-1)
    - [Confirm](#confirm-1)

---

## No `Cloudflare` DNS Records Update

### Issue

```txt
ExternalDNS deploys and runs healthy, but no DNS records appear in Cloudflare for `gitops-dev.arguswatcher.net`.

$ kubectl get pods -n external-dns
NAME                            READY   STATUS    RESTARTS   AGE
external-dns-59894cd57b-xjmkb   1/1     Running   0          8m
```

### Debugging

A running pod with no records means either

- (a) it can't talk to Cloudflare,
- (b) the API token lacks permissions,
- or (c) it isn't discovering any desired hostnames.

Check the logs first — they distinguish all three.

| Hypothesis             | Command                                                                                     | Verdict                       |
| ---------------------- | ------------------------------------------------------------------------------------------- | ----------------------------- |
| Auth / API failure     | `kubectl logs -n external-dns deploy/external-dns \| grep -iE "error\|forbidden\|401\|403"` | OK — no errors                |
| Provider misconfigured | `kubectl logs -n external-dns deploy/external-dns \| grep -i "provider"`                    | OK — `Provider:cloudflare`    |
| Domain filter wrong    | `kubectl logs ... \| grep -i DomainFilter`                                                  | OK — `[arguswatcher.net]`     |
| **Nothing to publish** | `kubectl logs ...` shows `All records are already up to date` on every loop                 | **FAIL — no desired records** |

```sh
kubectl logs -n external-dns deploy/external-dns --tail=5
time="..." level=info msg="All records are already up to date"
time="..." level=info msg="All records are already up to date"
```

> That message means ExternalDNS computed zero desired records — its sources returned nothing.

### Root cause

Two compounding issues in `external-dns.yaml`

1. **Sources only included `service`.** The hostname `gitops-dev.arguswatcher.net` is declared on the HTTPRoute (`apps/frontend/base/httproute.yaml`), not on any Service. `External DNS` won't see `HTTPRoute` hostnames unless `gateway-httproute` is in `sources`.
2. **No `external-dns.alpha.kubernetes.io/hostname` annotation** on the `NLB Service` either — so even with `service` as a source, there's nothing to map.

Confirm sources from the running config:

```sh
kubectl logs -n external-dns deploy/external-dns | grep -i "Sources:"
... Sources:[service] ...
```

### Fix

- Add `gateway-httproute` to the source list.
  - `ExternalDNS` will then walk every `HTTPRoute`, follow `parentRefs` to the `Gateway`, read the `Gateway`'s `status.addresses` (the NLB hostname), and create a CNAME per `HTTPRoute` hostname.

```yaml
# bootstrap/external-dns.yaml
sources:
  - service
  - gateway-httproute
```

### Confirm

- After Argo sync (pod restarts with new sources):

```sh
# Sources updated
kubectl logs -n external-dns deploy/external-dns | grep -i "Sources:"
# -> Sources:[service gateway-httproute]

# Records being created
kubectl logs -n external-dns deploy/external-dns -f
# -> Add records: gitops-dev.arguswatcher.net CNAME [...elb.ca-central-1.amazonaws.com]
# -> Add records: cname-gitops-dev.arguswatcher.net TXT [...]

# Verify in Cloudflare via API
ZONE_ID=$(curl -s -H "Authorization: Bearer $CF_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones?name=arguswatcher.net" \
  | jq -r '.result[0].id')
curl -s -H "Authorization: Bearer $CF_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records?name=gitops-dev.arguswatcher.net" \
  | jq '.result[] | {type, name, content}'

# End-to-end
curl -I https://gitops-dev.arguswatcher.net
# -> HTTP/2 200
```

- If logs still say "All records are already up to date" after ~2 minutes, the API token most likely lacks `Zone:DNS:Edit` + `Zone:Zone:Read` on `arguswatcher.net`.

---

## HTTPS returns "unable to get local issuer certificate"

### Issue

DNS resolves and the endpoint responds, but `curl` refuses the TLS handshake as untrusted.

```sh
curl https://gitops-dev.arguswatcher.net/
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

### Debugging

`curl: (60)` is always a trust-chain failure. Inspect the cert the server is actually presenting to find out who issued it.

| Hypothesis                         | Command                                                                     | Verdict                                          |
| ---------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------ |
| Local CA bundle stale              | `curl --cacert /etc/ssl/certs/ca-certificates.crt https://...`              | N/A — fails the same way                         |
| SNI mismatch                       | `openssl s_client -servername gitops-dev.arguswatcher.net -connect ...:443` | OK — SNI is correct                              |
| **Cert not issued by a public CA** | `openssl s_client ... \| openssl x509 -noout -subject -issuer -dates`       | **FAIL — Cloudflare Origin CA, not a public CA** |

```sh
echo | openssl s_client -connect gitops-dev.arguswatcher.net:443 \
    -servername gitops-dev.arguswatcher.net 2>/dev/null \
    | openssl x509 -noout -subject -issuer -dates
# subject=CN=CloudFlare Origin Certificate
# issuer=CloudFlare Origin SSL Certificate Authority
# notBefore=Jul  3 15:58:00 2025 GMT
# notAfter=Jun 29 15:58:00 2040 GMT
```

> The 15-year validity is the giveaway — public CAs cap at ~13 months. This is a Cloudflare Origin CA cert.

---

### Root cause

- The `ACM cert` wired into the NLB `platform/envoy/gateway.yaml` is a **Cloudflare Origin CA cert**, only trusted by Cloudflare's edge proxy — not by browsers or `curl`.
- It was designed for the `CF-edge` → origin leg, not for direct client → origin traffic. Because the Cloudflare DNS record was unproxied (grey cloud), clients hit the NLB directly and **saw the untrusted Origin cert**.

### Fix

Front the service with Cloudflare's proxy (orange cloud) so:

- Clients see Cloudflare's publicly-trusted edge cert.
- Cloudflare → NLB uses the Origin CA cert (which it trusts).
- End-to-end TLS, no further cert plumbing needed.

Two changes:

1. **ExternalDNS default**: add `--cloudflare-proxied` in `bootstrap/external-dns.yaml`

   ```yaml
   extraArgs:
     - --cloudflare-proxied
   ```

2. **Per-route override**: annotate the HTTPRoute in `apps/frontend/base/40_httproute.yaml`.

   ```yaml
   metadata:
     annotations:
       external-dns.alpha.kubernetes.io/cloudflare-proxied: "true"
   ```

3. **Cloudflare zone setting**: SSL/TLS → Overview → **Full (strict)**.
   - This makes CF validate the origin cert against the Origin CA — the Origin cert is only honored in this mode.

Note: `ExternalDNS` may not flip the proxied flag on a pre-existing record. If `dig` still returns AWS IPs after sync, delete the record in Cloudflare and let ExternalDNS recreate it.

### Confirm

- after Argo sync:

```sh
# Record now resolves to a Cloudflare IP, not AWS
dig +short gitops-dev.arguswatcher.net
# -> 104.x.x.x or 172.x.x.x (Cloudflare), not 15.x / 52.x (AWS)

# Cert is now publicly trusted
echo | openssl s_client -connect gitops-dev.arguswatcher.net:443 \
  -servername gitops-dev.arguswatcher.net 2>/dev/null \
  | openssl x509 -noout -issuer
# -> issuer=Google Trust Services / Let's Encrypt / similar

# End-to-end
curl -I https://gitops-dev.arguswatcher.net/
# -> HTTP/2 200
```
