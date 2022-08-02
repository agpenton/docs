---
title: "Staging"
date: 2022-07-17T13:35:47+02:00
draft: false
---

# Solving staging problems

## Master is deployed and it's not working

1. Check #dev-team channel if there announcement about someone working on staging.
2. If none, say "Component X is not working on staging, feature X is not working for me. Master is deployed"
3. Proceed to debug with "Known Issues" section
4. Check sentry for errors in broken component
5. Check kibana for errors in broken component (filter by app name)
6. Check kubectl logs for errors in broken component (`kubectl logs deploy/shop-app` ... etc )
7. Check Grafana dashboard for component on staging (set env to `juniqe-staging`)
8. CHeck last deployment job in gitlab for errors
9. Check AWS health dashboard if there are ongoing incidents

## You deployed some branch, it does not work

1. deploy master
2. If deployment  fails with errors "Waiting for pod default/runner-ldomxexz-project-13374321-concurrent-0wzvvr to be running, status is Pending
   ERROR: Job failed (system failure): prepare environment: timed out waiting for pod to start. Check https://docs.gitlab.com/runner/shells/index.html#shell-profile-loading for more information". follow "Gitlab CI Runner is not running".
2. Check if it's working
3. If master does not work, follow "Master is deployed and it's not working"
4. If master is working, sync your branch with master and then deploy

## You have not deployed anything and it does not work

1. Check if master is deployed
2. If not, Check #dev-team channel if there announcement about someone working on staging.
3. If not follow "You deployed some branch, it does not work"



## "Gitlab CI Runner is not running"

1. Try to restart Pipeline/job later
2. Check if other pipelines are also failing
3. How long pipelines are failing?
4. Check if there are many schedulle failures `kubectl get events -A | grep FailedScheduling`
5. Check if k8s have schedullable nodes. `kubectl get node | grep Ready` should show nodes (5-10 usually, not 0 and not 1)
6. If yes, proceed with k8s cluster cannot schedulle pods on nodes

## k8s cluster cannot schedulle pods on nodes

1. Check if spot.io controller is connected to spot.io. https://console.spotinst.com/spt/ocean/aws/kubernetes/view/o-fcc91800 Should not show any errors
2. Check spot.io Log tab for error "03/18/2021, 08:19:09, WARN, Auto-scaling activity is currently suspended due to a cluster configuration error. The spotinst cluster controller has not reported a heartbeat to the spotinst API."
3. If there are such error, check if controller is working `kubectl -n kube-system get deploy/spotinst-kubernetes-cluster-controller` should show ready status 1/1
4. Check logs for errors `kubectl -n kube-system logs deploy/spotinst-kubernetes-cluster-controller | grep -v INFO`
5. Try to reload spot.io controller `kubectl -n kube-system rollout restart deploy/spotinst-kubernetes-cluster-controller`
6. If nothing helps, contact spot.io support.



# Known Issues

## Shop project was freshly deplyoyed *after* nginx (blue whale on all pages)

This could happen if nginx was deployed before shop ingress

1. Check if staging.juniqe.com is showing blue whale on all pages(not only catalogue)
2. restart nginx-fronted service in AWS ECS console (ECS->backend-staging->services->nginx-frontend->update->force new deploy->skip to review-> apply )

## Shop Main page opens, but catalogue links are broken (blue whale)

Check sentry for errors like this

```
UnknownHostException com.juniqe.storage.rest.RestStorage in $anonfun$performRequest$1
error
storage6-01.juniqe-stagingcontrollers.shop.HealthController
```

search link https://sentry.io/organizations/juniqe/issues/?environment=juniqe-staging&project=1375821&query=is%3Aunresolved+storage&statsPeriod=24h

if storage cannot be reached, check

- ES storage is deployed, healthy, has indices and scripts
- app configuration is right
