# Debug: NLB curl hangs at TCP handshake

## Problem

`curl` to the NLB hostname hangs at the TCP handshake — connection never establishes.

```
$ curl -v -H "Host: gitops-dev.arguswatcher.net" http://gitops-demo-dev-420f0504fde951d4.elb.ca-central-1.amazonaws.com
*   Trying 15.222.78.91:80...
[hangs]
```

## Debug

NLB hang at TCP layer means no healthy targets, blocked SG, or wrong subnets. Rule each out in order.

| Hypothesis | Command | Verdict |
|---|---|---|
| App pod down / no endpoints | `kubectl get endpoints -n frontend gitops-demo-frontend` | OK — `10.0.11.205:80` |
| HTTPRoute not attached | `kubectl get gateway eg -n gateway -o jsonpath='{.status.listeners[*].attachedRoutes}'` | OK |
| NLB in private subnets | `aws ec2 describe-subnets --subnet-ids <ids> --query "Subnets[*].MapPublicIpOnLaunch"` | OK — all `true` |
| Node SG blocks NLB traffic | `aws ec2 describe-security-groups --group-ids <node-sg> --query "SecurityGroups[0].IpPermissions"` | OK — allows from NLB SG on `80–10443` |
| **Target group unhealthy** | `aws elbv2 describe-target-health --target-group-arn <tg-arn>` | **FAIL — `unhealthy / Target.FailedHealthChecks`** |

## Root cause

Health check probes the wrong port. Envoy data-plane listens on `10080`/`10443` (non-root), but the TG was configured to health-check `:80`.

```
$ aws elbv2 describe-target-groups --target-group-arns <tg-arn> \
    --query "TargetGroups[0].[HealthCheckProtocol,HealthCheckPort,Port]"
[ "HTTP", "80", 10080 ]   # probe port 80, but target listens on 10080
```

Driven by these annotations in [platform/envoy/gateway.yaml](../platform/envoy/gateway.yaml):

```yaml
service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: HTTP
service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: /
service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "80"
```

## Fix

Switch the health check to `traffic-port` so each TG probes its own listener port.

```yaml
# platform/envoy/gateway.yaml
service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: TCP
service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: traffic-port
```

Confirm after Argo sync (~30–60s for AWS-LBC to reconcile):

```
aws elbv2 describe-target-health --region ca-central-1 \
  --target-group-arn <tg-arn> \
  --query "TargetHealthDescriptions[*].TargetHealth.State"
# -> ["healthy"]

curl -I -H "Host: gitops-dev.arguswatcher.net" \
  http://gitops-demo-dev-420f0504fde951d4.elb.ca-central-1.amazonaws.com
# -> HTTP/1.1 200 OK
```
