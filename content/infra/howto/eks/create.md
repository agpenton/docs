---
title: "Create and update EKS Cluster with Pulumi"
date: 2022-07-18T19:03:11+02:00
draft: false
---

This Guide is for the creation or updating the AWS EKS Cluster with pulumi, hosted in one of our Environments.

### Requirementes: {style=text-align:left}
- [pulumi](https://www.pulumi.com/docs/get-started/install/)
- [NodeJS](https://nodejs.org/en/download/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/)
- [helm](https://helm.sh/docs/intro/install/)
- [helmsman](https://github.com/Praqma/helmsman)
- [helmfile](https://github.com/helmfile/helmfile)

### Steps:
{{% notice info %}}
Always remember that the login will last for 8 hours
{{% /notice %}}
- First you need to be login in AWS throught the terminal, more info [here](/infra/howto/environment/aws-cli-sso)
- After succefuly login, clone the repository where the source code it is:
```
git clone https://gitlab.com/juniqe/infra/cloud.git
```
- go to the eks directory:
```
cd cloud/aws/eks
```
- check that you are in the right directory:
```
ls -la
```
- the output of the previous command:
```
total 376
drwxr-xr-x   28 asdrubal  staff    896 Jul 18 15:37 .
drwxr-xr-x   17 asdrubal  staff    544 Feb 21 12:53 ..
-rw-r--r--@   1 asdrubal  staff     21 Feb 17 11:17 .gitignore
drwxr-xr-x    8 asdrubal  staff    256 Feb 17 11:17 .idea
-rw-r--r--@   1 asdrubal  staff    126 Feb 17 11:17 .prettierrc
-rw-r--r--@   1 asdrubal  staff   1130 Feb 17 11:17 JuniqeEKSProvider.ts
-rw-r--r--@   1 asdrubal  staff   2813 Feb 17 11:17 JuniqeEksCluster.ts
-rw-r--r--@   1 asdrubal  staff   3612 Jul 18 15:37 JuniqeEksNodegroupCluster.ts
-rw-r--r--@   1 asdrubal  staff   1068 Feb 17 11:17 Pulumi.juniqe-production-eks.yaml
-rw-r--r--    1 asdrubal  staff   1296 Jul 18 20:18 Pulumi.juniqe-staging-eks.yaml
-rw-r--r--@   1 asdrubal  staff     58 Feb 17 11:17 Pulumi.yaml
-rw-r--r--    1 asdrubal  staff  10732 Jul 15 09:09 index.ts
drwxr-xr-x  154 asdrubal  staff   4928 Feb 17 11:17 node_modules
-rw-r--r--    1 asdrubal  staff  47043 Feb 17 11:17 package-lock.json
-rw-r--r--    1 asdrubal  staff    296 Feb 17 11:17 package.json
-rw-r--r--    1 asdrubal  staff    340 Feb 17 11:17 tasks.py
-rw-r--r--    1 asdrubal  staff   1720 Jul 14 11:08 terraformState.ts
-rw-r--r--@   1 asdrubal  staff    438 Feb 17 11:17 tsconfig.json
```
- login in to the pulumi backend, in this case **juniqe-staging**:
```
pulumi login --cloud-url s3://juniqe-staging-pulumi
```
- Select the right stack depending on the environment that you are.
```
pulumi select juniqe-staging-eks
```
- install all the modules needed to apply the stack.
```
npm install
```
- test that the stack doesn't have any error:

{{% notice note %}}
You can check the stack everytime that you make a change.
{{% /notice %}}
```
pulumi preview
```
- if there is no issues, you can apply the stack:
```
pulumi up -y
```
{{% notice tip %}}
You can apply the stack as many times as you want, and you will always will get same results.
{{% /notice %}}
- The output of the preview command should be:
```
Previewing update (juniqe-staging-eks):
     Type                  Name                    Plan     Info
     pulumi:pulumi:Stack   eks-juniqe-staging-eks
     └─ eks:index:Cluster  juniqe                           1 warning

Diagnostics:
  eks:index:Cluster (juniqe):
    warning: The option 'ignoreChanges' has no effect on component resources.

Outputs:
  ~ instanceProfile              : {
        
   ... OUTPUT OMITED...
        
   }
```



 