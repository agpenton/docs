---
title: "Cloud"
date: 2022-07-17T13:31:51+02:00
draft: false
---

# [Monitoring](monitoring/README.md)

# [Using Staging](using-staging.md)

# [Solving staging issues](staging.md)

# Kubernetes

## Tooling

To work with AWS EKS Kubernetes you will need to install this tools:

- `invoke` command line utility to run our make-like tasks. This step is optional. [Learn more](#invoke)

- docker https://docs.docker.com/install/

- aws cli https://aws.amazon.com/cli/
- aws-iam-authenticator https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html
- amazon-ecr-credential-helper https://github.com/awslabs/amazon-ecr-credential-helper
- aws-google-auth https://github.com/cevoaustralia/aws-google-auth

* kubectl https://kubernetes.io/docs/tasks/tools/install-kubectl/
* skaffold https://skaffold.dev/docs/install/
* kustomize https://github.com/kubernetes-sigs/kustomize
* kubeseal https://github.com/bitnami-labs/sealed-secrets/releases (Optional)

## Preparation

Following steps are described for AWS juniqe-staging account

### AWS auth

We need to authorize in AWS to

- Get auth tokens for kubectl with aws-iam-authenticator
- to push docker images artifacts Elastic container Registry amazon-ecr-credential-helper

#### Setting up AWS profile first time run

Authorization in AWS is done via aws-google-auth, as our google account have mapping to aws roles

Open or create `~/.aws/config` file with following contents or add to existing one:

```
[profile juniqe-staging]
region = eu-central-1
google_config.ask_role = False
google_config.keyring = False
google_config.duration = 42000
google_config.google_idp_id = C00wp1uuf
google_config.google_sp_id = 86281751805
google_config.u2f_disabled = False
google_config.google_username = replace-username-with-yours@juniqe.com
google_config.role_arn = arn:aws:iam::445858552116:role/juniqe-staging/admin
google_config.bg_response = None

```

for production env:

```
[profile juniqe-production]
region = eu-central-1
google_config.ask_role = False
google_config.keyring = False
google_config.duration = 42000
google_config.google_idp_id = C00wp1uuf
google_config.google_sp_id = 86281751805
google_config.u2f_disabled = False
google_config.google_username = replace-username-with-yours@juniqe.com
google_config.role_arn = arn:aws:iam::054846004576:role/juniqe-production/operator
google_config.bg_response = None
```

Make sure you've replaced `replace-username-with-yours@juniqe.com` with yours juniqe email

Now run aws-google-auth command to authorize yourself in AWS via google account

```
aws-google-auth  -d 42000   -p juniqe-staging --role-arn arn:aws:iam::445858552116:role/juniqe-staging/admin
```

for production env

```
aws-google-auth  -d 42000   -p juniqe-production --role-arn arn:aws:iam::054846004576:role/juniqe-production/operator
```

By setting this env variable we will switch current aws profile to juniqe-staging. We should use it before running any AWS call

```
export AWS_PROFILE=juniqe-staging
```

For production, set to juniqe-production

```
export AWS_PROFILE=juniqe-production

```

To check if you are authorized in aws

```
aws sts get-caller-identity
```

You should get json with your account information

```
{
    "UserId": "AROAI4JI2N52XMITA3AIC:marat@juniqe.com",
    "Account": "445858552116",
    "Arn": "arn:aws:sts::445858552116:assumed-role/admin/marat@juniqe.com"
}

```

Confluence has more extensive info https://juniqe.atlassian.net/wiki/spaces/EN/pages/65568769/AWS+Credentials+via+SAML+SSO

#### ECR authorization

To authorize in ecr we need `aws-iam-authenticator` and docker helper configuration

Open `~/.docker/config.json` and add `credHelpers` object to json:

```
{
	"credHelpers":{ "603482864741.dkr.ecr.eu-central-1.amazonaws.com":"ecr-login"}
}
```

This will enable ECR auth trough amazon-ecr-credential-helper

### AWS EKS K8S authorization first time run

To authorize in K8S we have to provision KUBECONFIG file.

List EKS clusters available in AWS account

```
aws eks list-clusters

{
    "clusters": [
        "kjsmain-eksCluster-d71139f", ## old 1.14 cluster
        "juniqe-eksCluster-4e5ef16"  ## new 1.18+ cluster
    ]
}
```

Update your KUBECONFIG with new cluster:

```
aws eks update-kubeconfig --name juniqe-eksCluster-4e5ef16 --alias juniqe-staging
```

for production

```
aws eks update-kubeconfig --name juniqe-eksCluster-e5415f2 --alias juniqe-production
```

Switch to context and change default namespace

```
kubectl config use-context juniqe-staging
kubectl config set-context --current --namespace=juniqe-staging
```

Switch to context and change default namespace

```
kubectl config use-context juniqe-production
kubectl config set-context --current --namespace=juniqe-production
```

Now we can check if we have access to namespace:

```
kubectl get namespace
NAME                 STATUS   AGE
default              Active   197d
gitlab               Active   183d
juniqe-integration   Active   197d
juniqe-staging       Active   197d
kube-node-lease      Active   197d
kube-public          Active   197d
kube-system          Active   197d
```

Get pod list

```
kubectl get pod
NAME                                                   READY   STATUS       RESTARTS   AGE
busybox-7b5f4b66db-4c74c                               1/1     Running      0          31d
carts-api-extension-79dfb6587b-db5kn                   1/1     Running      0          13d
carts-api-extension-79dfb6587b-m7wxv                   1/1     Running      0          3d
carts-api-extension-79dfb6587b-n2wnh                   1/1     Running      0          2d21h
....
```

More info about

- EKS auth https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html
- KUBECONFIG https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/

## AWS EKS K8S everyday usage

Do AWS auth, which is valid for 8h

```
aws-google-auth  -d 42000   -p juniqe-staging --role-arn arn:aws:iam::445858552116:role/juniqe-staging/admin
```

For production

```
aws-google-auth  -d 42000   -p juniqe-production --role-arn arn:aws:iam::054846004576:role/juniqe-production/operator
```

Set AWS profile, which is valid in current terminal session

```
export AWS_PROFILE=juniqe-staging
# for production
# export AWS_PROFILE=juniqe-production
```

You are ready to use EKS

```
kubectl describe namespace
NAME                STATUS   AGE
default             Active   11d
gitlab              Active   11d
juniqe-production   Active   11d
kube-node-lease     Active   11d
kube-public         Active   11d
kube-system         Active   11d
```

To switch between different K8S cluster, please use `kubectl config` command

- Operator account has access only to one namespace

# Pulumi

## Read general info

https://www.pulumi.com/docs/intro/concepts/

## Install

https://pulumi.com/docs/reference/install/

## Usage

Go to Pulumi project directory (for example ./aws/eks or ./aws/iam)

```
cd ./aws/iam/
```

Install requirements

```
npm install
```

Make sure you have AWS_PROFILE set up in current session.

Login into stack state backend. There are different urls for staging and production. This is similiar to terraform state backend

```
pulumi login --cloud-url s3://juniqe-staging-pulumi
```

Select stack. Each project contains at lease production and staging stacks

```
pulumi stack ls
pulumi stack select iam-juniqe-staging
```

Check if you are ready to run pulumi

```
pulumi stack

```

If you want to apply changes, run:

```
pulumi up --diff
```

# Sealed Secrets

To protect secrets(passwords, api keys, other critical stuff) in gitlab we should use sealed secrets

More info: https://engineering.bitnami.com/articles/sealed-secrets.html

## Usage

For example our secret is DATABASE_PASSWORD with value `password`

- Install kubeseal

- encode secret value in base64

```
echo -n  "password" | base64
```

copy result and use in next step

- create temp file `secret.yaml` with secret resource

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: juniqe-app
data:
  DATABASE_PASSWORD: cGFzc3dvcmQ= # < this is base64 encoded `password`
```

- Make Sure you are authorized in K8S cluster and have access to it

- Create Sealed secret resource from secret. Make sure you've specified right namespace.

```
cat secret.yaml | kubeseal -n juniqe-staging --controller-name=secret-sealed-secrets  --controller-namespace=kube-system -o yaml
```

- Copy output with SealedSecret resource and put it into your k8s resource files.
  Make sure that you removed createTimestamp and namespace from top-level if it kustomize setup.

- apply your resources to k8s (with skaffold or kubectl) as usual

- SealedSecret controller will generate Secret based on your SealedSecret resource

- Remove `secret.yaml` file from your filesystem

- Deploy your service.

- check created secret:

```
kubectl get secret juniqe-app  -o json | jq .data.DATABASE_PASSWORD -r  | base64 -d
```

the answer should be the inital secret value ("password" in this case)

If Secret does not appear, please check SealedSecret controller pod logs

```
kubectl -n kube-system logs deploy/sealed-secrets
```

## Using this repo

This repo consists K8S cluster definition with components. The main cluster lives in aws/eks/ folder and written in pulumi.

K8S components are using root/component or root/category/component folder structure. Components are written k8s kind definitions, some of them are just downloaded and community managed helm charts. Some components definitions are written by us and using kustomize.

Installation of components and cluster are wrapped and unified by _invoke_.

### invoke

This is python cli tool similiar to make and fabric. invoke supports dependencies between tasks.

invoke uses `tasks.py` file to store tasks. tasks could have dependencies.

More info https://www.pyinvoke.org/

Each component has `tasks.py` file with default task to install or upgrade component.

### Installing invoke

ArchLinux:

```
pacman -Sy python-invoke
```

Or on any other distribution by using pip

```
pip install --user  invoke
```

### Install & upgrade component or cluster

To install any component, go to the folder of component and simply run `invoke` command. `invoke` will run default task to install or upgrade component using staging environment. To run on production `invoke install --env=juniqe-production`

### Installing everything

in the root of this repo, run `invoke` to run tasks from root tasks.py.

root tasks.py consists orchestration over creation & upgrading cluster configuration.

## Adding new commponents

Create folder on top level or in category/component
Add component via helm or other tool.
Wrap installation into invoke's tasks.py
Add task with dependencies to the root tasks.py


