---
title: "Performance"
date: 2022-07-17T18:53:30+02:00
draft: false
---

# How to address issues performance issues

Use this diagram to solve issues

https://drive.google.com/file/d/1WaTRu9OIoUy-4a9XbtqVbpYQpUpsfGoP/view?usp=sharing

## How to

### check if not enough resources

Memory: OOM events, memory usage is close to maximum, gc behaviour
CPU: High cpu usage(more or close then maximum)
Space: Service is non fuctional or throws errors on IO

### Check if we have more traffic then usually

Open Traffic rate grafics retrospectively for 1h, 12h, 2 days, 7 days windows and search for anomalies

### check if traffic spike is correlating with perormance issues

Look for spike window and correlate with reponse time, error rates

## Check errors

Check error rates on grafana metrics, for 5xx on shop and magento check kibana vis table for exact uri

`elk.intra.juniqe.com/_plugin/kibana/app/kibana#/visualize/edit/e71b0680-2361-11eb-bf46-11db10131e73?_g=h@44136fa&_a=h@518cb23`

check sentry, kibana app logs

### Is service in process of scaling

#### k8s deployment

For example for shop, check "Deployment pods"

kubectl describe hpa shop-app

current should be less then desired and going up

```
Deployment pods: 5 current / 5 desired
```

Also look at events field for scaling events or lack of them.

or check grafana dashboard for such metric

#### ECS service

Go to ECS console, open "backend" ecs cluster, find service and check for running / desired instances. Also check events tab for scaling events

or check datadog dashboard for such metric

### Cluster has enough capacity for new instances ?

Check events for instance allocation errors

```
kubectl describe hpa shop-app
```

Open AWS/EC2/Autoscaling page and check events tab for autoscaling events. There are should be no errors. Pools should not be at maximum

# Actions

## Vertical scaling (more resources)

### k8s deployment

Find k8s deployemnt resource in k8s definition and change `resources` specification.

```
          resources:
            requests:
              memory: "2600Mi"
              cpu: "2048m"
            limits:
              memory: "2600Mi"
              cpu: "4096m"
```

run `skaffold deploy -p juniqe-production --images "<current image tag>"`

### ECS

Find taskdefinition (for example juniqe-infra/terraform/modules/juniqe/ecs-shop/files/taskdefinition.json)

change `memory`, `memoryReservation`, `cpu`.

Run terraform init, terraform apply for this service

## Horizontal scaling (more instances)

### k8s deployement

Find k8s HorizontalPodAutoscaler resource and change maximum and minimum replicas

run `skaffold deploy -p juniqe-production --images "<current image tag>"`

### Ecs

Find service layer in juniqe-infra/terraform/envs and change maximum/minimum in service module.

# Main services

## Storefront CDN

Main content distribution. Proxies to nginx-frontend(shop,checkout) and pdp-app.

Caches /, /api/, `/_next/` and pdp-app pdp pages.

### Metrics

https://grafana.prod.juniqe.com/d/D9AMi9hGk/storefront-cdn?orgId=1

Check error rates

### Dependencies

- nginx-frontend
- pdp-app

## Nginx-frontend

Ingests and routes traffic between shop and magento

### Metrics

https://app.datadoghq.com/dashboard/628-s9p-pg2/nginx-frontend?from_ts=1605024289396&live=true&to_ts=1605027889396&tpl_var_env=juniqe-production

check cpu, amount of containers

### Logs

### Metrics

https://app.datadoghq.com/dashboard/628-s9p-pg2/nginx-frontend?from_ts=1605024289396&live=true&to_ts=1605027889396

### Dependencies

- shop-app
- magento-checkout

## Shop-app

### Metrics

check response time, error rates

https://grafana.prod.juniqe.com/d/HlhbEnKMz/shop-app?orgId=1

### Manual check

Page is opening https://www.juniqe.de/wandbilder/poster

Main page is cached, apis are cached

### Dependencies

- Masterdb
- Magento Internal
- ES Storage
- Caches (Redis & Memcached)
- DY

## PDP-APP

### Metrics

What to check: response time, error rates

https://grafana.prod.juniqe.com/d/FYp22acGz/pdp-app?orgId=1

### Manual check

pdp pages are all cached

You could try to open this pathes, which are not blocked
http://pdp-app.prod.juniqe.com/status

http://pdp-app.prod.juniqe.com/metrics

### Dependencies

- shop-app

## Magento Checkout

serves /checkout, payment provider callbacks

### Metrics

Check 5xx error rates, listen queue size, request times, container count

https://app.datadoghq.com/dashboard/j2s-r88-9gb/magento-checkout

### Manual check

Open Page https://www.juniqe.de/checkout/onepage/ or place an order with Advanced Payment

### Dependencies

- masterdb
- slavedb
- memcached (sessions)
- redis (configuration)

## Magento Internal

serves /reboot/ endpoints which are used in shop /cart/add /cart/get and session generation.

### Metrics

Check 5xx error rates, listen queue size, request times, container count

https://app.datadoghq.com/dashboard/i9n-aph-uyk/magento-internal

### Manual check

Open https://www.juniqe.de/reboot/session/init
Should generate new session if there are no frontent session cookie

### Dependencies

- masterdb
- slavedb
- memcached (sessions)
- redis (configuration)

## Master Database

### Metrics

https://grafana.prod.juniqe.com/d/7afX4nKGz/amazon-mysql-rds?orgId=1

Check CPU, Burst Balance

### Manual check

Connect to database

```
mysql -h masterdb.juniqe-production
```

run `select now()`

Check processlist for long queries

```
select ID,USER,TIME,STATE from information_schema.processlist where COMMAND!="Sleep" order by TIME\G
```

### Vertical scaling

In ~/workspace/juniqe-infra/terraform/envs/juniqe-production/env.auto.tfvars change "instance.types.rds" and apply

## Slave database

Special replica is used only during blackfriday to unload some read only magento checkout queries.

### Metrics

https://grafana.prod.juniqe.com/d/7afX4nKGz/amazon-mysql-rds?orgId=1

To find slow queries, this tool can be used
AWS performance insights https://eu-central-1.console.aws.amazon.com/rds/home?region=eu-central-1#performance-insights:resourceId=db-OMUTTKKBJDQDDHOEOYI3W66D2I;resourceName=masterdb

Check Replica Lag, CPU, Burst Balance

### Vertical scaling

In ~/workspace/juniqe-infra/terraform/envs/juniqe-production/env.auto.tfvars change "instance.types.rds" and apply

### Horizontal scaling

In `~/workspace/juniqe-infra/terraform/envs/juniqe-production/rds-slavedb/main.tf`

add new slave module with uniqe name (slavedb-02,03..etc).
make sure `load_weight` is set to 0, as we do not want to send traffic to not warmed up instance.

run `terraform apply`

After successfull apply (spinning new instance will take 10-20 min), start increasing load_weight gradually with 10-20 step each 10-20 min.

Monitor cpu and query latency, query distribution skew between slaves

### Manual check

Connect to database

```
mysql -h slavedb-01.juniqe-production
```

Check processlist for long queries, or if there are any queries

```
select ID,USER,TIME,STATE from information_schema.processlist where COMMAND!="Sleep" order by TIME\G
```

check lag

```
SHOW SLAVE STATUS\G
```

There are should be no errors, Seconds_Behind_Master: 0 and Slave_SQL_Running: Yes

any lag can stop checkout from happening. there are automation to route queries to master

check current routing:

dig default.rr.readonly-slavedb.juniqe-production

as this record is server side round robin, it can return different result each run if weight is not 0% or 100%

## ElasticSearch Storage

### Metrics

Check:

- cluster status,
- shards in non active state should be zero (affecting shop availability)
- threadpools & queues (large queue could mean long shop api response)
- thread pool rejections ( non zero could mean failed queries)
- heap

https://grafana.prod.juniqe.com/d/n_nxrE_mk/juniqe-storage-elasticsearch?orgId=1&refresh=1m&var-env=juniqe-production

### Manual check

Check cluster status is green:

http://storage6-01.juniqe-production:9200/_cluster/health

Check which indice is broken:
http://storage6-01.juniqe-production:9200/_cat/indices?v

Check nodes status:
http://storage6-01.juniqe-production:9200/_cat/nodes?v

Use ES head chrome extension to debug cluster and indices health
https://chrome.google.com/webstore/detail/elasticsearch-head/ffmkiejjmecolpfloofpjologoblkegm

### Horizontal scaling

in '~/workspace/juniqe-infra/terraform/envs/juniqe-production/storage/main.tf'

change `elasticsearch_cluster_size` (on-demand instances) or `elasticsearch_cluster_spot_size`.

run `terraform apply`

Do not add too much instances at once.
