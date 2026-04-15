---
title: "Atlantis on AWS EKS with FluxCD and GitHub"
description: "A practical guide to deploying Atlantis on AWS EKS using FluxCD for GitOps delivery and integrating it with a GitHub Personal or Organization account for Terraform pull request automation."
summary: "Deploy Atlantis on AWS EKS using FluxCD and connect it to GitHub to automate Terraform plan and apply workflows through pull requests."
date: 2024-01-01T00:00:00+00:00
lastmod: 2024-01-01T00:00:00+00:00
draft: false
weight: 10
toc: true
seo:
  title: ""
  description: ""
  canonical: ""
  noindex: false
---

## What is Atlantis and Why Should You Care?

If you have ever been on a team where running `terraform apply` felt like a game of roulette — who ran it last, from which machine, against which workspace — you already know the pain Atlantis solves.

[Atlantis](https://www.runatlantis.io/) is an open-source tool that brings Terraform automation directly into your pull request workflow. Instead of engineers running Terraform locally or through a fragile CI script, Atlantis listens for GitHub (or GitLab/Bitbucket) webhook events, runs `terraform plan` automatically when a PR touches `.tf` files, and lets you trigger `terraform apply` with a simple comment — all in the context of a code review.

The result is **collaborative, auditable, and safe infrastructure changes**. Every plan is visible to the whole team. Every apply is tied to an approved pull request. No more "who ran that?" conversations at 2 AM.

This guide walks you through deploying Atlantis on an **AWS EKS** cluster using **FluxCD** for GitOps-style delivery, and wiring it up to a **GitHub Personal or Organization** repository.

---

## Architecture Overview

Before we dive in, here is what we are building:

```
GitHub Repo (terraform code)
       │
       │  webhook (push/PR events)
       ▼
   Atlantis Pod
   (running in EKS)
       │
       │  terraform plan/apply
       ▼
  AWS Infrastructure
```

And the delivery pipeline for Atlantis itself:

```
GitHub Repo (atlantis manifests)
       │
       │  GitOps sync
       ▼
    FluxCD
   (running in EKS)
       │
       │  reconciles HelmRelease / manifests
       ▼
   Atlantis Pod
   (running in EKS)
```

FluxCD owns the deployment of Atlantis into the cluster. GitHub owns the Terraform workflow events. Atlantis sits in the middle and does the work.

---

## Prerequisites

Make sure you have the following in place before starting:

- An **AWS EKS cluster** up and running (EKS 1.27+ recommended)
- **kubectl** configured and pointing at your cluster (`kubectl get nodes` should work)
- **FluxCD** bootstrapped in your cluster — if not, see [FluxCD Bootstrap](https://fluxcd.io/flux/installation/)
- **Helm** installed locally (v3+)
- A **GitHub Personal Access Token (PAT)** or a **GitHub App** with repo and webhook permissions
- A domain or an AWS Load Balancer URL for the Atlantis webhook endpoint
- **ExternalDNS** or manual DNS — Atlantis needs a publicly reachable HTTPS endpoint

> **Note on HTTPS:** GitHub will only deliver webhooks to HTTPS endpoints. If you are setting this up for the first time, the easiest path is an AWS ALB with an ACM certificate in front of Atlantis. We will set up the Service as `LoadBalancer` type and annotate it accordingly.

---

## Step 1 — Prepare GitHub

### Personal Repository

For a personal account, you need two things: a **Personal Access Token** and a **Webhook**.

**Create a PAT:**

1. Go to **GitHub → Settings → Developer Settings → Personal Access Tokens → Tokens (classic)**
2. Generate a new token with the following scopes:
   - `repo` (full control of private repositories)
   - `read:org` (if you want Atlantis to read team membership)
3. Save the token — you will not see it again

**Generate a Webhook Secret:**

```shell
openssl rand -hex 32
```

Save this value — you will use it in both the GitHub webhook configuration and the Atlantis deployment.

**Create the Webhook (do this after Atlantis is deployed and has a public URL):**

1. Go to your repository → **Settings → Webhooks → Add webhook**
2. Set **Payload URL** to `https://your-atlantis-domain/events`
3. Set **Content type** to `application/json`
4. Paste your webhook secret
5. Select **Let me select individual events** and check:
   - Pull requests
   - Issue comments
   - Push
6. Save

### Organization Repository

For a GitHub Organization, the setup is identical, but you create the webhook at the organization level if you want Atlantis to handle multiple repos:

1. Go to **GitHub Organization → Settings → Webhooks**
2. Same webhook configuration as above — but now it fires for all repos in the org

For the PAT, use an organization member's token with `repo` scope, or — better for production — create a **GitHub App** scoped to the organization. GitHub Apps are more granular and do not tie credentials to a personal account.

> **Recommendation for Org setups:** Use a GitHub App over a PAT. It gives you per-repo install control, does not expire (unlike fine-grained tokens), and does not die if an employee leaves. The Atlantis docs cover [GitHub App configuration](https://www.runatlantis.io/docs/access-credentials.html#github-app) in detail.

---

## Step 2 — Store Secrets in Kubernetes

Atlantis needs two sensitive values: the GitHub token and the webhook secret. Never hardcode these in manifests. We will create a Kubernetes Secret and reference it from the Helm values.

```shell
kubectl create namespace atlantis

kubectl create secret generic atlantis-secrets \
  --namespace atlantis \
  --from-literal=github-token=ghp_YOUR_TOKEN_HERE \
  --from-literal=webhook-secret=YOUR_WEBHOOK_SECRET_HERE
```

If you are using **AWS Secrets Manager** with the **External Secrets Operator**, create an `ExternalSecret` instead:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: atlantis-secrets
  namespace: atlantis
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: atlantis-secrets
    creationPolicy: Owner
  data:
    - secretKey: github-token
      remoteRef:
        key: prod/atlantis
        property: github-token
    - secretKey: webhook-secret
      remoteRef:
        key: prod/atlantis
        property: webhook-secret
```

Using External Secrets is the production-grade approach — credentials rotate automatically and never live in your Git history.

---

## Step 3 — Configure FluxCD to Deploy Atlantis

FluxCD works by reconciling the state of your cluster against manifests stored in Git. We will add a `HelmRelease` for Atlantis to your FluxCD repository.

### Add the Helm Repository Source

In your FluxCD config repository (the one FluxCD is bootstrapped against), create the following file:

```yaml
# clusters/my-cluster/atlantis/helmrepository.yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: runatlantis
  namespace: flux-system
spec:
  interval: 12h
  url: https://runatlantis.github.io/helm-charts
```

### Create the HelmRelease

```yaml
# clusters/my-cluster/atlantis/helmrelease.yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: atlantis
  namespace: atlantis
spec:
  interval: 10m
  chart:
    spec:
      chart: atlantis
      version: "5.x"   # pin to a major version, check https://artifacthub.io/packages/helm/runatlantis/atlantis
      sourceRef:
        kind: HelmRepository
        name: runatlantis
        namespace: flux-system
      interval: 12h
  values:
    orgAllowlist: "github.com/YOUR_GITHUB_ORG_OR_USERNAME/*"

    github:
      user: "your-github-username-or-bot"
      token:
        secretKeyRef:
          name: atlantis-secrets
          key: github-token
      secret:
        secretKeyRef:
          name: atlantis-secrets
          key: webhook-secret

    # Expose Atlantis via a LoadBalancer (ALB via AWS Load Balancer Controller)
    service:
      type: LoadBalancer
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: "external"
        service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
        service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"

    # Resource requests — Atlantis is lightweight
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"

    # Persistent storage for Terraform working directories
    storageClassName: "gp2"
    dataStorage: "5Gi"

    # Repo config — inline or via ConfigMap (see Step 4)
    repoConfig: |
      repos:
        - id: github.com/YOUR_GITHUB_ORG_OR_USERNAME/YOUR_REPO
          allow_custom_workflows: true
          allowed_overrides:
            - workflow
            - apply_requirements
          apply_requirements:
            - approved
            - mergeable
```

> **Important:** Replace `YOUR_GITHUB_ORG_OR_USERNAME` and `YOUR_REPO` with your actual values. The `orgAllowlist` controls which repos Atlantis will respond to — keep it as specific as possible.

### Commit and Push

```shell
git add clusters/my-cluster/atlantis/
git commit -m "feat(atlantis): add HelmRepository and HelmRelease for Atlantis"
git push
```

FluxCD will pick this up within its reconciliation interval (default 10 minutes) and deploy Atlantis into the `atlantis` namespace. Watch it reconcile:

```bash
flux get helmreleases -n atlantis --watch
```

---

## Step 4 — Configure Repository-Level Workflows

Atlantis uses a `repos.yaml` (server-side) and optionally an `atlantis.yaml` (per-repo) to define how Terraform projects are discovered and executed.

### Server-Side `repos.yaml`

This is what we embedded inline in the `HelmRelease` above. For more complex setups, mount it as a ConfigMap:

```yaml
repos:
  - id: /.*/   # match all allowed repos
    allow_custom_workflows: true
    allowed_overrides:
      - workflow
      - apply_requirements
    apply_requirements:
      - approved       # PR must have at least one approval
      - mergeable      # PR must have no merge conflicts

workflows:
  default:
    plan:
      steps:
        - init
        - plan
    apply:
      steps:
        - apply
```

### Per-Repo `atlantis.yaml`

Place this file at the root of each Terraform repository:

```yaml
# atlantis.yaml
version: 3
automerge: false
delete_source_branch_on_merge: false
projects:
  - name: networking
    dir: ./networking
    workspace: default
    terraform_version: v1.7.0
    autoplan:
      when_modified:
        - "*.tf"
        - "*.tfvars"
      enabled: true
    apply_requirements:
      - approved

  - name: eks-cluster
    dir: ./eks
    workspace: default
    terraform_version: v1.7.0
    autoplan:
      when_modified:
        - "*.tf"
      enabled: true
    apply_requirements:
      - approved
```

This tells Atlantis exactly which directories to treat as Terraform projects, which files trigger a plan, and what version of Terraform to use per project.

---

## Step 5 — Point the Webhook at Atlantis

Once the `HelmRelease` is reconciled and the pod is running, get the external address of the LoadBalancer:

```shell
kubectl get svc atlantis -n atlantis
# NAME       TYPE           CLUSTER-IP     EXTERNAL-IP                          PORT(S)
# atlantis   LoadBalancer   10.100.x.x     abc123.elb.us-east-1.amazonaws.com   80:30xxx/TCP,443:30xxx/TCP
```

If you are using ExternalDNS, it will automatically create a DNS record pointing your domain at this address. Otherwise, create a CNAME in Route53 manually:

```
atlantis.yourdomain.com → abc123.elb.us-east-1.amazonaws.com
```

Then go back to your GitHub webhook (from Step 1) and set the Payload URL to:

```
https://atlantis.yourdomain.com/events
```

---

## Step 6 — Test the Integration

Open a pull request that modifies a `.tf` file in a monitored directory. Within seconds you should see:

1. Atlantis posts a comment on the PR with the output of `terraform plan`
2. After the PR is approved, comment `atlantis apply` to trigger the apply
3. Atlantis posts the apply output and locks the workspace to prevent concurrent runs

If the plan comment does not appear, check the Atlantis logs:

```bash
kubectl logs -n atlantis -l app.kubernetes.io/name=atlantis --tail=100 -f
```

Common issues:

| Symptom | Likely Cause |
|---|---|
| No comment on PR | Webhook not firing — check GitHub webhook delivery logs |
| `repo not allowlisted` error | `orgAllowlist` in Helm values does not match the repo |
| `Error: could not find atlantis.yaml` | Add an `atlantis.yaml` to the repo root, or set `autoDiscover` in server config |
| `Error acquiring lock` | Another plan/apply is in progress for the same workspace |
| Plan succeeds but apply fails with permissions error | The IAM role attached to the Atlantis pod does not have the required permissions |

---

## IAM Permissions for Atlantis on EKS

Atlantis needs AWS credentials to run Terraform. The recommended approach on EKS is **IRSA (IAM Roles for Service Accounts)**.

1. Create an IAM role with the permissions Terraform needs (e.g., `AdministratorAccess` for a dev cluster, or a tightly scoped policy for production)
2. Trust the EKS OIDC provider:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/oidc.eks.REGION.amazonaws.com/id/OIDC_ID"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.REGION.amazonaws.com/id/OIDC_ID:sub": "system:serviceaccount:atlantis:atlantis"
        }
      }
    }
  ]
}
```

3. Annotate the Atlantis ServiceAccount in the Helm values:

```yaml
serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/atlantis-terraform-role
```

This way, Atlantis pods automatically receive short-lived AWS credentials via the pod identity mechanism — no static keys, no secrets rotation headaches.

---

## Security Hardening Checklist

Before going to production, run through this list:

- [ ] `orgAllowlist` is as specific as possible — no wildcards unless intentional
- [ ] `apply_requirements` includes `approved` and `mergeable`
- [ ] Webhook secret is set and validated by Atlantis
- [ ] Atlantis pod is running as a non-root user (`securityContext.runAsNonRoot: true`)
- [ ] The Atlantis namespace has a `NetworkPolicy` restricting egress to only GitHub and AWS APIs
- [ ] IRSA is used — no static AWS credentials anywhere
- [ ] Atlantis is not accessible from the internet except on port 443 for webhook delivery
- [ ] GitHub PAT (or App) has the minimum required permissions
- [ ] FluxCD is configured to alert on `HelmRelease` failures

---

## Wrapping Up

You now have a fully GitOps-delivered Atlantis deployment on EKS, integrated with GitHub, and ready to take Terraform collaboration to the next level. Every infrastructure change goes through a pull request, every plan is visible and reviewable, and every apply is tied to an approval.

A few things to explore next:

- **Atlantis + Terragrunt** — if you use Terragrunt for DRY configs, Atlantis has native support for it via custom workflows
- **Drift Detection** — pair Atlantis with a scheduled Terraform plan job (e.g., a CronJob in Kubernetes) to detect configuration drift between applies
- **Multi-workspace support** — use `workspace` in `atlantis.yaml` to manage staging and production with a single PR workflow

The beauty of this setup is that once it is running, it largely runs itself. PRs trigger plans, approvals trigger applies, and FluxCD keeps Atlantis itself up to date. That is the DevOps dream — infrastructure that manages itself.

---

## References

- [Atlantis Documentation](https://www.runatlantis.io/docs/)
- [Atlantis Helm Chart](https://artifacthub.io/packages/helm/runatlantis/atlantis)
- [FluxCD Documentation](https://fluxcd.io/flux/)
- [AWS EKS IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [External Secrets Operator](https://external-secrets.io/)
