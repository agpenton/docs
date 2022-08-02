---
title: "Checkout app"
date: 2022-07-16T13:26:06+02:00
draft: false
---

## Checkout cart

**staging**: https://www.staging.juniqe.com/cm/#/

**production**: https://www.juniqe.de/cm/

**Logs**: https://sentry.io/organizations/juniqe/issues/?project=5272619

### Run checkout tests

Run in browser from localhost

```
npm run test:local
```

Run in headless Chrome from localhost

```
npm run test:local:headless
```

Run in browser against Staging

```
test:juniqe-staging
```

```
npx testcafe chrome qa/tests/functional.js --selector-timeout 50000 --assertion-timeout 20000 --page-load-timeout 15000 -e
```

The preferred domain can be selected by `--domain=de` or `domain=com`

### local dev with shop integration

1. Start local dev env with shop and nginx-frontend, for example

```
cd ~/workspace/juniqe-infra/docker
docker-compose up -d
```

2. Start checkout app

```
npm install
npm start
```

3. Open http://www.localhost.juniqe.de/cm/#/

nginx-frontend passes all traffic from "/cm" to localhost:3000/cm

### build

```
npm install
npm run build:juniqe-staging
```

### deploy

get aws autorization

```
export AWS_PROFILE=juniqe-staging
npm run deploy:juniqe-staging
```

### Versioning

version is branch(or tag) + sha

version is available here: cm/\_version.txt

For example: https://www.staging.juniqe.com/cm/_version.txt

## infrastructure

checkout app related infratructure is described in **infra** directory with [Pulumi](https://www.pulumi.com/docs/)


To change infrastructure:

1. Install Pulumi https://www.pulumi.com/docs/get-started/install/
2. Go to infra subfolder `cd infra/`
3. Install pulumi deps: `npm install`
4. Login to state backed `pulumi login --cloud-url s3://juniqe-staging-pulumi`
5. Select stack `pulumi stack select juniqe-staging-checkout-app`
6. To apply changes: `pulumi up --diff`

## Monitoring lambda

[Documentation](qa/README.md)
