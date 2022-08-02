---
title: "DevOps Entrypoints"
date: 2022-07-15T13:43:31+02:00
draft: false
---

###  New accounts [on/off] boarding

- **VPN management**: [Production VPN](https://gitlab.com/juniqe/juniqe-infra/-/blob/master/docker/infra/openvpn-server/README.md)
- **MySQL users**: 
- **SSO**: 

### Providers
- **Cognito** - no actions are needed.
- **Commercetools MC SSO**: [CommerceTools SSO](https://gitlab.com/juniqe/commercetools/project-setup/-/blob/master/docs/sso.md).

### Google Groups.
On GSuite new Engineering accounts should be added to these groups:
- All
- Engineering
For DevOps also the group:
- DevOps

GitLab and Sentry: access should be granted to the projects

### Infrastructure management
New infrastructure on K8s is documented in: [cloud repository](https://gitlab.com/juniqe/infra/cloud/)
- Everything around K8s
- DNS management for the cluster
- K8s IAM
- ALB for K8s
- BI infrastructure

The old part of the infrastructure is in: [juniqe-infra](https://gitlab.com/juniqe/juniqe-infra/)
- Env creation & global configs
- VPC & Peering
- Frontend LB
- Emarsys subdomains
- DNS outside of K8s
- Certificates outside of K8s
- RDS DB
- Elastic Caches
- Other team support services
- Local setup & docker containers

Production to staging data migration: 

#### MySQL
https://gitlab.com/juniqe/juniqe-infra/-/tree/master/docker/infra/rds-manager
https://gitlab.com/juniqe/juniqe-infra/-/blob/master/terraform/modules/juniqe/rds-master/main.tf#L25
https://gitlab.com/juniqe/juniqe-infra/-/blob/master/terraform/modules/juniqe/rds-master/main.tf#L17

#### ES
https://gitlab.com/juniqe/infra/storage
https://gitlab.com/juniqe/infra/storage/-/blob/master/k8s/base/curatorJobs.yaml
The backups are stored in S3 we should implement a retention policy for the backups.

#### VPC Peering
https://gitlab.com/juniqe/juniqe-infra/-/blob/master/terraform/envs/juniqe-staging/vpc/production_vpc_peering.tf
https://gitlab.com/juniqe/juniqe-infra/-/blob/master/terraform/envs/juniqe-production/vpc/staging_vpc_peering.tf
https://app.clickup.com/4580628/v/dc/4bt8m-736/4bt8m-47881

### DNS Management
![Infra](https://t4580628.p.clickup-attachments.com/t4580628/ea5f6656-44b3-4846-871e-9dac6bd37660/image.png "Infra")

The DNS management for all shop domains is done as part of the PDP app, in: https://gitlab.com/juniqe/pdp-app/-/tree/master/dns

The configuration of the DNS per application is done as part of the app infra.

Route53/Domain management is part of the juniqe-main account and is managed manually.

We also have 2 zones: prod.juniqe.com and stage.juniqe.com that's managed in: https://gitlab.com/juniqe/infra/cloud/-/tree/master/external-dns

Email sub-domains are managed in terraform: https://gitlab.com/juniqe/juniqe-infra/-/tree/master/terraform/envs/juniqe/marketing


### Weak points and caution zones:
nginx-frontend and CloudFront

### Logging:
[Logging](https://app.clickup.com/4580628/v/dc/4bt8m-736/4bt8m-33021)

### K8s IAM policy injection
https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html
https://gitlab.com/juniqe/logistics/-/blob/master/infra/logistic-app/iam.ts
https://gitlab.com/juniqe/logistics/-/blob/master/k8s/base/resources/LogisticApp.yaml#L49
https://gitlab.com/juniqe/logistics/-/blob/master/k8s/base/resources/LogisticAppSA.yaml


### Load Balancers:
**Controller:** https://gitlab.com/juniqe/infra/cloud/-/tree/master/aws/aws-load-balancer-controller

### CDN
- **Storefront CDN**: https://gitlab.com/juniqe/pdp-app/

#### Image resize:

![Image resize:](https://t4580628.p.clickup-attachments.com/t4580628/e0167cca-8562-4b1f-913f-32139a7c6b7d/images_rendering_v3.drawio.png "Image resize")

[//]: # ({{< mermaid >}})

[//]: # (sequenceDiagram)

[//]: # (    participant Browser)

[//]: # (    participant Cloudfront)

[//]: # (    participant S3)

[//]: # (    participant API Gateway)

[//]: # (    participant Lambda)

[//]: # (    Browser->>Cloudfront: GET)

[//]: # (    Cloudfront->>S3: GET  lookup image on S3 Backend)

[//]: # (    S3->>Cloudfront: GET lookup image on S3 Backend)

[//]: # (    loop Healthcheck)

[//]: # (    John->John: Fight against hypochondria)

[//]: # (    end)

[//]: # (    Note right of John: Rational thoughts <br/>prevail...)

[//]: # (    John-->Alice: Great!)

[//]: # (    John->Bob: How about you?)

[//]: # (    Bob-->John: Jolly good!)

[//]: # ({{< /mermaid >}})


**Image resize:**
- https://gitlab.com/juniqe/juniqe-infra/-/tree/master/terraform/modules/juniqe/lambda-image-rendering
- https://juniqe.atlassian.net/wiki/spaces/EN/pages/50757697/Rendering+API
- https://juniqe.atlassian.net/wiki/spaces/EN/pages/504135681/Rendering+Sequence
- https://bitbucket.org/juniqe/juniqe-infra/src/SYS-746-terraform-our-lambdaapi-gateway-/terraform/envs/README.md

**Checkout:** https://gitlab.com/juniqe/checkout-app/