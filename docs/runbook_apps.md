# Runbook - Debug Applications

[Back](../README.md)

- [Runbook - Debug Applications](#runbook---debug-applications)
  - [Debug Connection](#debug-connection)
  - [Argo Rollouts](#argo-rollouts)

---

## Debug Connection

```sh
# forward backend
kubectl port-forward svc/gitops-backend 8080:8080 -n backend

# forward frontend
kubectl port-forward svc/gitops-frontend 8800:80 -n frontend

# test against load balancer dns
curl -v -H "Host: gitops-dev.arguswatcher.net" http://gitops-demo-dev-420f0504fde951d4.elb.ca-central-1.amazonaws.com
curl -v -H "Host: gitops-stage.arguswatcher.net" http://gitops-stage-bfb16a510327f164.elb.ca-central-1.amazonaws.com
```

---

## Argo Rollouts

```sh
kubectl argo rollouts get rollout gitops-backend -n backend
# Name:            gitops-backend
# Namespace:       backend
# Status:          ✖ Degraded
# Message:         RolloutAborted: Rollout aborted update to revision 13
# Strategy:        Canary
#   Step:          0/5
#   SetWeight:     0
#   ActualWeight:  0
# Images:          docker.io/simonangelfong/gitops-demo-backend:v0.3.1 (stable)
# Replicas:
#   Desired:       4
#   Current:       4
#   Updated:       0
#   Ready:         4
#   Available:     4

# NAME                                        KIND        STATUS        AGE    INFO
# ⟳ gitops-backend                            Rollout     ✖ Degraded    3h34m
# ├──# revision:13
# │  └──⧉ gitops-backend-859b8f8985           ReplicaSet  • ScaledDown  9m33s  canary
# ├──# revision:12
# │  └──⧉ gitops-backend-6bb69bd6bb           ReplicaSet  ✔ Healthy     109m   stable
# │     ├──□ gitops-backend-6bb69bd6bb-5j6s2  Pod         ✔ Running     47m    ready:1/1
# │     ├──□ gitops-backend-6bb69bd6bb-qbmkj  Pod         ✔ Running     45m    ready:1/1
# │     ├──□ gitops-backend-6bb69bd6bb-qm5mt  Pod         ✔ Running     7m20s  ready:1/1
# │     └──□ gitops-backend-6bb69bd6bb-bqrl6  Pod         ✔ Running     2m31s  ready:1/1
# ├──# revision:9
# │  └──⧉ gitops-backend-7f499f5d45           ReplicaSet  • ScaledDown  99m
# ├──# revision:8
# │  └──⧉ gitops-backend-8d84d58c4            ReplicaSet  • ScaledDown  101m
# ├──# revision:7
# │  └──⧉ gitops-backend-c46546f75            ReplicaSet  • ScaledDown  102m
# ├──# revision:5
# │  └──⧉ gitops-backend-cc684c667            ReplicaSet  • ScaledDown  141m
# ├──# revision:4
# │  └──⧉ gitops-backend-77d8688d4b           ReplicaSet  • ScaledDown  3h18m
# ├──# revision:3
# │  └──⧉ gitops-backend-5c484454cd           ReplicaSet  • ScaledDown  3h23m
# └──# revision:2
#    └──⧉ gitops-backend-5884f6c9d8           ReplicaSet  • ScaledDown  3h30m

kubectl describe rollout gitops-backend -n backend
# Events:
#   Type     Reason                Age                    From                 Message
#   ----     ------                ----                   ----                 -------
#   Normal   ScalingReplicaSet     50m                    rollouts-controller  Scaled up ReplicaSet gitops-backend-7f499f5d45 (revision 9) from 1 to 4
#   Normal   ScalingReplicaSet     49m                    rollouts-controller  Scaled up ReplicaSet gitops-backend-6bb69bd6bb (revision 10) from 0 to 1
#   Normal   RolloutUpdated        49m                    rollouts-controller  Rollout updated to revision 10
#   Normal   RolloutNotCompleted   49m                    rollouts-controller  Rollout not completed, started update to revision 10 (6bb69bd6bb)
#   Normal   ScalingReplicaSet     49m                    rollouts-controller  Scaled down ReplicaSet gitops-backend-7f499f5d45 (revision 9) from 4 to 3
#   Normal   RolloutPaused         48m (x10 over 3h23m)   rollouts-controller  Rollout is paused (CanaryPauseStep)
#   Normal   RolloutStepCompleted  48m (x7 over 3h23m)    rollouts-controller  Rollout step 1/5 completed (setWeight: 25)
#   Normal   RolloutStepCompleted  47m (x5 over 3h22m)    rollouts-controller  Rollout step 2/5 completed (pause: 60s)
#   Normal   RolloutResumed        47m (x6 over 3h22m)    rollouts-controller  Rollout is resumed
#   Normal   RolloutNotCompleted   10m                    rollouts-controller  Rollout not completed, started update to revision 11 (859b8f8985)
#   Normal   RolloutUpdated        10m                    rollouts-controller  Rollout updated to revision 11
#   Normal   NewReplicaSetCreated  10m                    rollouts-controller  Created ReplicaSet gitops-backend-859b8f8985 (revision 11)
#   Normal   ScalingReplicaSet     10m                    rollouts-controller  Scaled down ReplicaSet gitops-backend-6bb69bd6bb (revision 10) from 4 to 3
#   Normal   ScalingReplicaSet     10m                    rollouts-controller  Scaled up ReplicaSet gitops-backend-859b8f8985 (revision 11) from 0 to 1
#   Normal   RolloutUpdated        7m49s                  rollouts-controller  Rollout updated to revision 12
#   Normal   SkipSteps             7m49s                  rollouts-controller  Rollback to stable ReplicaSets
#   Normal   ScalingReplicaSet     2m59s (x2 over 7m48s)  rollouts-controller  Scaled up ReplicaSet gitops-backend-6bb69bd6bb (revision 12) from 3 to 4
#   Warning  RolloutAborted        2m59s                  rollouts-controller  ReplicaSet "gitops-backend-859b8f8985" has timed out progressing.
#   Warning  RolloutAborted        2m59s                  rollouts-controller  Rollout aborted update to revision 13

kubectl get events -n backend --sort-by='.lastTimestamp'
# 5m35s       Warning   Unhealthy              pod/gitops-backend-859b8f8985-qskbq      Readiness probe failed: HTTP probe failed with statuscode: 503
# 5m35s       Warning   Unhealthy              pod/gitops-backend-859b8f8985-qskbq      Liveness probe failed: HTTP probe failed with statuscode: 503
# 5m24s       Warning   Unhealthy              pod/gitops-backend-859b8f8985-qskbq      Readiness probe failed: Get "http://10.0.12.216:8080/api/healthz": dial tcp 10.0.12.216:8080: connect: connection refused
# 4m30s       Normal    Started                pod/gitops-backend-859b8f8985-qskbq      Container started
# 4m30s       Normal    Created                pod/gitops-backend-859b8f8985-qskbq      Container created
# 4m30s       Normal    Pulled                 pod/gitops-backend-859b8f8985-qskbq      Container image "docker.io/simonangelfong/gitops-demo-backend:v0.3.1" already present on machine and can be accessed by the pod
# 4m30s       Normal    Killing                pod/gitops-backend-859b8f8985-qskbq      Container gitops-backend failed liveness probe, will be restarted
# 3m39s       Normal    SuccessfulDelete       replicaset/gitops-backend-859b8f8985     Deleted pod: gitops-backend-859b8f8985-qskbq
# 3m39s       Normal    SuccessfulCreate       replicaset/gitops-backend-6bb69bd6bb     Created pod: gitops-backend-6bb69bd6bb-bqrl6
# 3m39s       Normal    Killing                pod/gitops-backend-859b8f8985-qskbq      Stopping container gitops-backend
# 3m39s       Normal    Started                pod/gitops-backend-6bb69bd6bb-bqrl6      Container started
# 3m39s       Normal    Created                pod/gitops-backend-6bb69bd6bb-bqrl6      Container created
# 3m39s       Normal    Pulled                 pod/gitops-backend-6bb69bd6bb-bqrl6      Container image "docker.io/simonangelfong/gitops-demo-backend:v0.3.1" already present on machine and can be accessed by the pod
# 3m39s       Normal    Scheduled              pod/gitops-backend-6bb69bd6bb-bqrl6      Successfully assigned backend/gitops-backend-6bb69bd6bb-bqrl6 to ip-10-0-12-184.ca-central-1.compute.internal
# 3m39s       Normal    ScalingReplicaSet      rollout/gitops-backend                   Scaled up ReplicaSet gitops-backend-6bb69bd6bb (revision 12) from 3 to 4
# 3m39s       Warning   RolloutAborted         rollout/gitops-backend                   ReplicaSet "gitops-backend-859b8f8985" has timed out progressing.
# 3m39s       Warning   RolloutAborted         rollout/gitops-backend                   Rollout aborted update to revision 13
# 3m27s       Warning   Unhealthy              pod/gitops-backend-6bb69bd6bb-bqrl6      Readiness probe failed: Get "http://10.0.12.218:8080/api/healthz": dial tcp 10.0.12.218:8080: connect: connection refused

```
