---
title: "Cloud & Infrastructure"
date: 2022-07-15T13:43:31+02:00
draft: false
---
-----------
## Environments

We run our servers in different environments (separate AWS accounts). These are:

- juniqe-production
- juniqe-staging
- juniqe-integration (currently not in use)

## Deployments
We follow the following motos for all our deployments:
- Application infrastructure should be managed and deployed as part of the application code.
- All deployments should be implemented with CI/CD in mind.
- FreatureBranching should be automated and be part of the CI/CD.


Any change to our applications is currently going through the following steps:
- Feature branch creation.
- Feature implementation.
- Once the MR is created the CI is triggered for the MR.
- Code Review.
- QA (as needed).
- CD once the MR is approved by the Code Owner and the QA.

## Critical Services.
The Juniqe Infrastructure is composed of several services, including the lambdas. In all those services, exist some that are more critical than others, this is the list of them:
- **carts-API-extension:** (*add to cart API response*).
- **storage** (*ELK stack where the index of the products are hosted*).
- **shop** (*the main service of the shop*).
- **Magento** (*backend of the shop*).
- **PDP** (*the service that allows the customer to customize the products*).