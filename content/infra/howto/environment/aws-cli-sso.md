---
title: "Login using aws-cli in Terminal"
date: 2022-07-18T18:59:47+02:00
draft: true
---

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
    "Arn": "arn:aws:sts::445858552116:assumed-role/admin/replace-username-with-yours@juniqe.com"
}

```