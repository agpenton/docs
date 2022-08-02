---
title: "System"
date: 2022-07-17T18:48:22+02:00
draft: false
---

# Tools

## Grafana

Grafana https://grafana.prod.juniqe.com/d/XjxcnIcMk/juniqe?orgId=1

grafana is main tool for dashboarding and alerting. Grafana uses prometheus, aws cloudwatch, aws X-ray, Kibana and other datasources as timeseries source  to draw panels and fire metircs.

Main datasource is named "Prometheus thanos". This datasource provides most of the metrics via PromQL api and combines prometheus metrics from juniqe-staging and juniqe-production.


## PromQL - prometheus Query language.

We have to use PromQL to query prometheus timeseries metric storage, configure panels.

Examples:

Return all time series with the metric http_requests_total:

`http_requests_total`

Return all time series with the metric http_requests_total and the given job and handler labels:

`http_requests_total{job="apiserver", handler="/api/comments"}`

Return a whole range of time (in this case 5 minutes) for the same vector, making it a range vector:

`http_requests_total{job="apiserver", handler="/api/comments"}[5m]`

Note that an expression resulting in a range vector cannot be graphed directly, but viewed in the tabular ("Console") view of the expression browser.

Using regular expressions, you could select time series only for jobs whose name match a certain pattern, in this case, all jobs that end with server:

`http_requests_total{job=~".*server"}`

All regular expressions in Prometheus use RE2 syntax.

To select all HTTP status codes except 4xx ones, you could run:

`http_requests_total{status!~"4.."}`


Docs:

* Query Basics: https://prometheus.io/docs/prometheus/latest/querying/basics/
* Cheat Sheet: https://promlabs.com/promql-cheat-sheet/
* PromQL in Grafana:  https://grafana.com/docs/grafana/latest/datasources/prometheus/

To get list available metrics, got to explore https://grafana.prod.juniqe.com/explore  and start typing roughly what you need. Grafana will suggest possible metrics.

### well-known metrics prefixes

* `aws_` - imported AWS cloudwatch metrics
* `aws_lambda_` - aws lambda metrics
* `aws_sqs_` - aws sqs metrics
* `prometheus_` - prometheus internal metrics
* `juniqe_` - Custom juniqe app metrics
* `juniqe_shop_` - custom juniqe_shop metrics
* `juniqe_ct_` - commercetools imported metric (funnel-monitoring lambda)
* `http_` - http services counters, histograms (also used in juniqe apps)
* `kube_` - k8s related metrics
* `container_` - container related metrics

### well-known metrics labels


* `env` - juniqe Environment. Value can be juniqe-staging or juniqe-production
* `namespace` - kubernetes namespace
* `name` - exact ID of entity
* `tag_Something` - imported from aws resource tag. It is preffered to use tags over IDs or names or ARNs





## Sentry

https://sentry.io/organizations/juniqe/projects/


Sentry is application error managing tools  with smart event merging capabilities. Every app has it's own project in sentry. Sentry usually only recieve **error** or higher level  events such as logged errors and exceptions.

## ELK aka Kibana

https://elk.intra.juniqe.com/\_plugin/kibana/app/kibana#/home?\_g=()

Kibana contains:

* mostly all logs from all applications(except lambdas)
* nginx access logs

Kibana implements easy to use logs filtering and analyzing solution

Same data is avaliable in Grafana from elasticsearch datasource

## AWS cloudwatch

AWS monitoring solution. Provides many different products for metrics, logs and tracing

Can be accessed via https://eu-central-1.console.aws.amazon.com/cloudwatch/home?region=eu-central-1

Full lambda log, eks plane logs can  be found in cloudwatch logs

Logs and metrics are available from AWS cloudwatch datasource. Metrics from datasource are lack of tags.

Metrics are also available from prometheus datasource under `aws_` prefix.

## AWS Xray

AWS application tracing solution. Mainly used in lambdas to trace performance.

X-ray is available in grafana as datasource

# Morning triage workflow

- Triage new errors in grafana sentry and anomalies in grafana before standup
1. Sentry: Open your project with filters Environment=juniqe-production time=24h (or time since last triaging) with sort by frequency and lookup for new err without assignee and issue
2. Look at grafana dashboard if there are some possible anomaly
- Give team a small overview about new issues in your scope on standup
    1. New errors or High frequent errors
    2. Alerts
    3. Anomlies
- create tickets on issus if needed
- Solve issue if it's High freq, user facing error. possible high conversion decrease or notify person who can do that
- Apply hotfix and close issue


# What to do with alert

* notify team that you are checking this problem
* Go to Grafana Panel Dashboard by link in Alert
* Check if this error creates problems for end users
* Check for other symptoms on Dashboard (RPS, Errors)
* Check logs in sentry, cloudwatch(available in grafana)
* Find root cause, create an issue
* Start working fix if this is urgent

# Application - person responsibility matrix

How to use this: put 1 for first responsibilty and 2 for second. Check checksums on the edges. For each project put dashboard link and sentry link

https://docs.google.com/spreadsheets/d/1EXsZjphL7Yc5_-t6IWTS05lNYJg_sJQMqSDkAXQ9nrw/edit#gid=0

# How to monitor

1. Ensure that application have sentry integrated, put Link to sentry into _Application - person responsibility matrix_ doc and Readme.md of project
2. Ensure that k8s service application has prometheus /metrics published (scala, nodejs apps) and PodMonitor k8s resouce is configured
3. Ensure that lambda has X-ray integrated
4. Ensure that lambda stack resources (lambda, sqs, api gw), have tag Name (they would not appear in prometheus otherwise)
5. Create grafana dashboard. Examples
    - Lambda: https://grafana.prod.juniqe.com/d/i0_BHJ5Gz/cart-public-api-lambda?orgId=1
    - k8s service app: https://grafana.prod.juniqe.com/d/HlhbEnKMz/shop?orgId=1

# Which alerts to create and what monitor

## Alert on High Latency

Mandatory alert

Reflects resposivness of service.

### Examples:

* Execution time of lambda
* http response time latency


### How to set threshold:

500ms-1000ms - services involved in customer engagement (shop, category, pdp-app, etc), before conversion.
1000ms-5000ms - services after conversion: checkout, returns

Exact value should be set by historical p90 or p95 when service was normally functioning.

Alternatively, better practice is to use advanced  APDEX score: https://prometheus.io/docs/practices/histograms/#apdex-score https://en.wikipedia.org/wiki/Apdex

## Alert on number of Errors

Mandatory alert

Amount of errors. This metric describes overall health of application

### Examples:

* lambda not successful invocations
* 5xx errors
* 4xx errors


### How to set threshold

Best case scenarion alert has to be set for `max(errors{5m}) > 0`.
If app throws errors as part of normal functioning (404 not found, intermediate errors), threshold should be set for historical p95.

Alternatives:

* Calcualte percent of errors
* Include errors in APDEX score as Frustrated

## Alert on number of requests

Not Mandatory alert.

Amount of requests.

### Examples:

* http requests per second(hour,minute) to application
* new messages in sqs per second

### How to set threshold

historical maximum * 2

## Alert on resource saturation

Not Mandatory alert


Aim to monitor and track the usage of every resource the service relies upon. Some resources have hard limits you cannot exceed, like RAM, disk, or CPU quota allocated to your application. Other resources—like open file descriptors, active threads in any thread pools, waiting times in queues, or the volume of written logs—may not have a clear hard limit but still require management.

Depending on the programming language in use, you should monitor additional resources:

    In Java: The heap and metaspace size, and more specific metrics depending on what type of garbage collection you’re using
    In Go: The number of goroutines

The languages themselves provide varying support to track these resources.

    When the resource has a hard limit
    When crossing a usage threshold causes performance degradation

You should have monitoring metrics to track all resources—even resources that the service manages well. These metrics are vital in capacity and resource planning.

Examples:

* Lambda throttles
* Disk usage

### How to set threshold

5-10% of total resource


