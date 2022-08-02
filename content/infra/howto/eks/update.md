---
title: "Create and update EKS Cluster with Pulumi (continue)"
date: 2022-07-18T21:37:28+02:00
draft: true
---

This is continuation of the preview [guide](/infra/howto/eks/create)

{{% notice note %}}
To upgrade the cluster you should know that kubernetes don't allow skip version. So you will need to upgrade one by one, ex:
**from 1.18 to 1.22** you need to go throught **1.19**, **1.20**, **1.21**, **1.22**. Every upgrade take about 45 minutes.
{{% /notice %}}

- To make the upgrade you need to change the version of the cluster in the pulumi stack:
```
vim Pulumi.juniqe-staging-eks.yaml
```
- modify the version number to the next one, ex: **"1.21"**
```
...
version: "1.21"
...
```
- after the change in the config file, check what are the new changes added to the stack, to do so run:
```
pulumi preview
```
the output of that command should similar to this:

```
Previewing update (juniqe-staging-eks):
     Type                               Name                                                                    Plan        Info
     pulumi:pulumi:Stack                eks-juniqe-staging-eks
     ├─ eks:index:Cluster               juniqe                                                                              1 warning
 ~   │  ├─ aws:eks:Cluster              juniqe-eksCluster                                                       update      [diff: ~version]
 +-  │  ├─ aws:ec2:LaunchConfiguration  juniqe-nodeLaunchConfiguration                                          replace     [diff: ~imageId]
 ~   │  └─ aws:cloudformation:Stack     juniqe-nodes                                                            update      [diff: ~templateBody]
     ├─ eks:index:NodeGroup             juniqe-eksCluster-4e5ef16-t4g-medium-on-demand
 +-  │  ├─ aws:ec2:LaunchConfiguration  juniqe-eksCluster-4e5ef16-t4g-medium-on-demand-nodeLaunchConfiguration  replace     [diff: ~imageId]
 ~   │  └─ aws:cloudformation:Stack     juniqe-eksCluster-4e5ef16-t4g-medium-on-demand-nodes                    update      [diff: ~templateBody]
     └─ eks:index:NodeGroup             juniqe-eksCluster-4e5ef16-t3-medium-on-demand
 +-     ├─ aws:ec2:LaunchConfiguration  juniqe-eksCluster-4e5ef16-t3-medium-on-demand-nodeLaunchConfiguration   replace     [diff: ~imageId]
 ~      └─ aws:cloudformation:Stack     juniqe-eksCluster-4e5ef16-t3-medium-on-demand-nodes                     update      [diff: ~templateBody]

Diagnostics:
  eks:index:Cluster (juniqe):
    warning: The option 'ignoreChanges' has no effect on component resources.

Outputs:
  ~ clusterVersion               : "1.20" => "1.21"
  ~ instanceProfile              : {
        arn       : "arn:aws:iam::445858552116:instance-profile/role-5fb6438"
        createDate: "2021-02-12T14:58:19Z"
        id        : "role-5fb6438"
        name      : "role-5fb6438"
        path      : "/"
        role      : "juniqe-instanceRole-role-5c6b951"
        uniqueId  : "AIPAWPTZ5TE2CLMXEG4YH"
        urn       : "urn:pulumi:juniqe-staging-eks::eks::aws:iam/instanceProfile:InstanceProfile::role"
    }
  ~ instanceRole                 : {
        arn                : "arn:aws:iam::445858552116:role/juniqe-instanceRole-role-5c6b951"
        assumeRolePolicy   : (json) {
            Statement: [
                [0]: {
                    Action   : "sts:AssumeRole"
                    Effect   : "Allow"
                    Principal: {
                        Service: "ec2.amazonaws.com"
                    }
                }
            ]
            Version  : "2012-10-17"
        }

        createDate         : "2021-01-05T14:17:09Z"
        forceDetachPolicies: false
        id                 : "juniqe-instanceRole-role-5c6b951"
        inlinePolicies     : [
            [0]: {
                name  : "return-pdf-dhl-DHL-invoke"
                policy: (json) {
                    Statement: [
                        [0]: {
                            Action  : [
                                [0]: "execute-api:Invoke"
                                [1]: "execute-api:ManageConnections"
                            ]
                            Effect  : "Allow"
                            Resource: "arn:aws:execute-api:eu-central-1:445858552116:red0n9efff/*/GET/return/*"
                        }
                    ]
                    Version  : "2012-10-17"
                }

            }
        ]
        managedPolicyArns  : [
            [0]: "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
            [1]: "arn:aws:iam::aws:policy/AdministratorAccess"
            [2]: "arn:aws:iam::aws:policy/AmazonSNSFullAccess"
            [3]: "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
            [4]: "arn:aws:iam::aws:policy/AmazonS3FullAccess"
            [5]: "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
            [6]: "arn:aws:iam::aws:policy/AWSLambda_FullAccess"
            [7]: "arn:aws:iam::aws:policy/AWSSecurityHubFullAccess"
            [8]: "arn:aws:iam::aws:policy/AWSCloudFormationFullAccess"
        ]
        maxSessionDuration : 3600
        name               : "juniqe-instanceRole-role-5c6b951"
        path               : "/"
        uniqueId           : "AROAWPTZ5TE2EGJ6GB27W"
        urn                : "urn:pulumi:juniqe-staging-eks::eks::eks:index:Cluster$eks:index:ServiceRole$aws:iam/role:Role::juniqe-instanceRole-role"
    }
```