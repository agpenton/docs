---
title: "AWS SSO via Google SAML"
date: 2022-07-15T13:43:31+02:00
draft: false
---
Many information is available in Internet and detailed manual configuration described in AWS Blog
At JUNIQE configuration made via TF for each account.

### AWS config
The main AWS account configuration at the moment located at:
- https://bitbucket.org/juniqe/system-tools/src/master/terraform-2.0/providers/aws/global/main.tf?at=master&fileviewer=file-view-default

With one resource description:
    ```
    # SAML Providerresource "aws_iam_saml_provider" "juniqe" {  name                   = "GSuite"  saml_metadata_document = "${file("files/saml-metadata.xml")}"}
    ```
After this we create a roles which is mapping in Google Admin with custom schema.

### Google config

In section of Apps > SAML Apps is always available the latest metadata for the Google IdP which is used in terraform.

### Custom Schema

We need to build a mapping for users in Google and AWS.
For this one better to extend existing schema with custom fields.
That is the Schema used for user:

```
{
  "fields":
          [
            {
            "fieldName": "role",
            "fieldType": "STRING",
            "readAccessType": "ADMINS_AND_SELF", 
            "multiValued": true
            },
            {
            "fieldName": "duration",
            "fieldType": "INT64",
            "readAccessType": "ADMINS_AND_SELF"
            }
          ],
  "schemaName": "SSO"
}
```
Could be added via developers.google.com: schemas/insert

### Mapping between AWS and Google.
2 options is required, but with 3rd option we can extend the session time.

| APP setting | Provider section   | Provider attribute         |
| ------ |--------------------|----------------------------|
| https://aws.amazon.com/SAML/Attributes/RoleSessionName | Basic Information. | Primary Email <sup>*</sup> |
| https://aws.amazon.com/SAML/Attributes/Role | SSO.               | role <sup>**</sup>         |
| https://aws.amazon.com/SAML/Attributes/SessionDuration   | SSO.               | duration                   |

<sup>*</sup> this attribute is responsible what name of user will be in the AWS logs

<sup>**</sup> that is the list of available AWS Roles for user

### Assign user via Google Admin

Via the Google Admin panel is possible to add/change all custom attributes.
As an example:
- The role attribute is consist of "<role ARN>,<provider ARN>". Provider ARN changes per AWS account (and provider but we are using only one at the moment)

### Assign user, alternatives

It could be done with developers.goohle.com users/patch, or any custom script, which changes permissions for users. But via Google Admin it is easier anyway.
Here an example of payload with all roles for our accounts at the moment.

```
{
  "customSchemas": {
    "SSO": {
      "duration": 43200,
      "role": [
        {
          "value": "arn:aws:iam::603482864741:role/juniqe/admin,arn:aws:iam::603482864741:saml-provider/google",
          "customType": "JUNIQE Main Admin"
        },
        {
          "value": "arn:aws:iam::603482864741:role/juniqe/readonly,arn:aws:iam::603482864741:saml-provider/google",
          "customType": "JUNIQE Main ReadOnly"
        },
        {
          "value": "arn:aws:iam::530590334745:role/juniqe-datavirtuality/admin,arn:aws:iam::530590334745:saml-provider/google",
          "customType": "DataVirtuality Admin"
        },
        {
          "value": "arn:aws:iam::530590334745:role/juniqe-datavirtuality/readonly,arn:aws:iam::530590334745:saml-provider/google",
          "customType": "DataVirtuality ReadOnly"
        },
        {
          "value": "arn:aws:iam::097140904496:role/snowplow-juniqe/admin,arn:aws:iam::097140904496:saml-provider/google",
          "customType": "SnowPlow Admin"
        },
        {
          "value": "arn:aws:iam::097140904496:role/snowplow-juniqe/readonly,arn:aws:iam::097140904496:saml-provider/google",
          "customType": "SnowPlow ReadOnly"
        },
        {
          "value": "arn:aws:iam::445858552116:role/juniqe-staging/admin,arn:aws:iam::445858552116:saml-provider/google",
          "customType": "juniqe-staging Admin"
        },
        {
          "value": "arn:aws:iam::445858552116:role/juniqe-staging/readonly,arn:aws:iam::445858552116:saml-provider/google",
          "customType": "juniqe-staging ReadOnly"
        },
        {
          "value": "arn:aws:iam::603482864741:role/juniqe/designers,arn:aws:iam::603482864741:saml-provider/google",
          "customType": "JUNIQE Main designers"
        },
        {
          "value": "arn:aws:iam::603482864741:role/juniqe/finance,arn:aws:iam::603482864741:saml-provider/google",
          "customType": "JUNIQE Main finance"
        }
      ]
    }
  }
}

```