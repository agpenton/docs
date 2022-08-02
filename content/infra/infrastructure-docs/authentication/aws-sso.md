---
title: "AWS Credentials via SAML (SSO)"
date: 2022-07-15T13:43:31+02:00
draft: false
---

If you want to have AWS API access for any script or terraform  or even just AWS CLI outside of AWS you need to have a working AWS credential.
Making and maintenance of such credentials could be a problem and big security issue.
Luckily with login via Google IDP, we don't have anymore permanent keys.
The problem that engineers don't have API access. As a solution - we can assumeRoleWith SAML to obtain the AWS STS token. This temporary credentials will work for 1 hour.

> Starting from 0.0.24 longer session (up to 12h) and SAML caching is supported by aws-google-auth 

### How to use aws-google-auth with our docker terraform toolbox

- Clone or pull last juniqe-infra *git@gitlab.com:juniqe/juniqe-infra.git*
- Go to terraform folder

```shell
cd ~/workspace/juniqe-infra/terraform
```
- Make sure you have "~/.aws" folder  with proper rights first 

```
# mkdir ~/.aws
# touch ~/.aws/config ~/.aws/credentials ~/.aws/saml_cache.xml
# ls -la ~/.aws/
total 32
drwx------  2 maratsh users  4096 Aug 23 14:49 ./
drwx------ 67 maratsh users  4096 Aug 24 12:00 ../
-rw-------  1 maratsh users  2400 Aug 23 16:20 config
-rw-------  1 maratsh users 10883 Aug 23 16:20 credentials
-rw-------  1 maratsh users  7709 Aug 23 16:20 saml_cache.xml
```

- Open ~/.aws/config and create configs per environment. replace profile and username.

```
[profile replace me, in example: juniqe-staging]
region = eu-central-1
google_config.ask_role = False
google_config.keyring = False
google_config.duration = 42000
google_config.google_idp_id = C00wp1uuf
google_config.google_sp_id = 86281751805
google_config.u2f_disabled = False
google_config.google_username = replace me, in example: marat@juniqe.com
 
 
[profile replace me, in example: datavirtuality]
region = eu-central-1
google_config.ask_role = False
google_config.keyring = False
google_config.duration = 42000
google_config.google_idp_id = C00wp1uuf
google_config.google_sp_id = 86281751805
google_config.u2f_disabled = False
google_config.google_username = replace me, in example: marat@juniqe.com
```
- Run aws-google-auth from terraform toolbox container. I will ask you for juniqe google password and probably for captcha. Try to open it in your browser. Enter your MFA token or press Yes in Google app on your phone. Now you can select available aws account from dropdown menu by using number.

```
# docker-compose run -e AWS_PROFILE="replace me, in example: juniqe-staging" --rm terraform aws-google-auth -a -p replace me, in example: juniqe-staging
Failed to import U2F libraries, U2F login unavailable. Other methods can still continue.
Google Password:
https://accounts.google.com/Captcha?v=2&ctoken=AAWk9lSauaGz-ckoVCJ7eSDsnzm4jTwUQB79C4N1jhGKjSgW7r4XIfdEbtM29siisGWnCgFDZkc14R_Srp518xFVRHHR3Afb02jXhoq1-MuLoRsTn3YrqSEVNASVD_nBOJQ9jNcMLUfTdjJ_CylUPJv-NWF7Nu1mOipczySGYwDlH3x1JPwMg58
Captcha (case insensitive): crouseracy
Open the Google App, and tap 'Yes' on the prompt to sign in ...
[  1] arn:aws:iam::445858552116:role/juniqe-staging/admin
[  2] arn:aws:iam::445858552116:role/juniqe-staging/readonly
Type the number (1 - 2) of the role to assume: 7
Assuming arn:aws:iam::445858552116:role/juniqe-staging/admin
Credentials Expiration: 2018-08-24 00:31:22+00:00
```

- Check that you successfully authorized 

```
# docker-compose run -e AWS_PROFILE=juniqe-staging(replace it with your profile name) --rm terraform aws --profile juniqe-staging(replace it with your profile name) sts get-caller-identity
 
{
    "UserId": "AROAI4JI2N52XMITA3AIC:marat@juniqe.com",
    "Account": "445858552116",
    "Arn": "arn:aws:sts::445858552116:assumed-role/admin/marat@juniqe.com"
}
```

#### Authorize on multiple envs at once

Example of the script to get multiple credentials at once (only first call need to be authorized)

Uncomment what you need and place it somewhere in PATH, in example ~/.bin/

ex: `juniqe-aws-auth.sh`
```
#!/bin/bash
set -uxe
cd ~/workspace/juniqe-infra/terraform
docker-compose run --rm terraform aws-google-auth  -d 42000   -p juniqe-staging --role-arn arn:aws:iam::445858552116:role/juniqe-staging/readonly
docker-compose run --rm terraform aws-google-auth  -d 42000   -p juniqe-integration --role-arn  arn:aws:iam::067547835157:role/juniqe-integration/readonly
 
docker-compose run --rm terraform aws-google-auth  -d 42000   -p juniqe-staging --role-arn arn:aws:iam::445858552116:role/juniqe-staging/admin
docker-compose run --rm terraform aws-google-auth  -d 42000   -p juniqe-integration --role-arn  arn:aws:iam::067547835157:role/juniqe-integration/admin
 
docker-compose run --rm terraform aws-google-auth  -d 42000   -p juniqe-production --role-arn arn:aws:iam::054846004576:role/juniqe-production/admin
docker-compose run --rm terraform aws-google-auth  -d 42000   -p juniqe-main --role-arn arn:aws:iam::603482864741:role/juniqe/admin
```

#### Examples of running
all commands should be running `workspace/juniqe-infra/terraform`

- Init ( vendors and state download ), plan (dry-run)  and apply(actual run) cms on juniqe-staging

```
# Init cms-app layer:
docker-compose run -e AWS_PROFILE="juniqe-staging" -w /tf/envs/juniqe-staging/cms-app  --rm terraform tf init
# plan layer with variable version=1.0.0:
docker-compose run -e AWS_PROFILE="juniqe-staging" -w /tf/envs/juniqe-staging/cms-app  --rm terraform tf plan -var version=1.0.0
# Apply layer
docker-compose run -e AWS_PROFILE="juniqe-staging" -w /tf/envs/juniqe-staging/cms-app  --rm terraform tf apply -var version=1.0.0
```
- Run [terragrunt](https://github.com/gruntwork-io/terragrunt) plan on whole juniqe-integration infrastructure via terragrunt.

```
docker-compose run -e AWS_PROFILE="juniqe-integration" --rm terraform terragrunt plan-all --terragrunt-non-interactive --terragrunt-working-dir /tf/envs/juniqe-integration
```
- get all instances on juniqe-staging via  aws cli.

```
docker-compose run -e AWS_PROFILE="juniqe-staging" terraform aws ec2 describe-instances
```