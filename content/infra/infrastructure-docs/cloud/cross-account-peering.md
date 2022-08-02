---
title: "AWS private zone cross-account peering"
date: 2022-07-15T13:43:31+02:00
draft: false
---

### HOWTO

#### Requirements:

- AWS Account or Role access.
- aws-google-auth installed
- docker and docker-compose installed.
- Git installed.

It's possible to resolve internal zone records from one vpc to another in different account.
*For example:*
We have the account `juniqe-production` with zone `juniqe-production` (zone id **ZONEIDJPR**) and `juniqe-main` account
with vpc `vpc-7main`; The goal is to make zone `juniqe-production` resolvable inside `juniqe-main`.
To do so:

- Get the repository **juniqe-infra** for Gitlab:
    ```
    git clone https://gitlab.com/juniqe/juniqe-infra.git
    ```
  then, and go to terraform directory:
    ```
    cd terraform
    ```
- Get authorization for aws profile `juniqe-main` and `juniqe-production`.
    ```
    aws-google-auth  -d 42000 -a  -p  juniqe-production
    aws-google-auth  -d 42000 -a  -p  juniqe-main
    ```
- Authorize **juniqe-main vpc** to associate with zone `juniqe-production` in `juniqe--production` account.
    ```
    docker-compose run -e AWS_PROFILE="juniqe-production" --rm terraform aws route53 create-vpc-association-authorization --hosted-zone-id ZONEIDJPR --vpc "VPCRegion=eu-central-1,VPCId=vpc-7main"
        
    {
        "HostedZoneId": "ZONEIDJPR",
        "VPC": {
            "VPCRegion": "eu-central-1",
            "VPCId": "vpc-7main"
        }
    }
    ```
- Create association in juniqe-main account.
  ```
  docker-compose run -e AWS_PROFILE="jsmain" --rm terraform   aws route53 associate-vpc-with-hosted-zone --hosted-zone-id ZONEIDJPR --vpc "VPCRegion=eu-central-1,VPCId=vpc-7main"
  
  {
  "ChangeInfo": {
     "Id": "/change/C2S4FIJCTH1OEL",
     "Status": "PENDING",
     "SubmittedAt": "2018-07-06T14:40:41.032Z",
     "Comment": ""
    }
  }
  ```
- Done; Check the DNS inside `juniqe-main`; Authority section should be filled.
  ```
  dig juniqe-production 
  
  ; <<>> DiG 9.13.0 <<>> juniqe-production
  ;; global options: +cmd
  ;; Got answer:
  ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 45662
  ;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1
  
  ;; OPT PSEUDOSECTION:
  ; EDNS: version: 0, flags:; udp: 4096
  ;; QUESTION SECTION:
  ;juniqe-production. IN A
  
  ;; AUTHORITY SECTION:
  juniqe-production. 60 IN SOA ns-1536.awsdns-00.co.uk. awsdns-hostmaster.amazon.com. 1 7200 900 1209600 86400

  ;; Query time: 21 msec
  ;; SERVER: 172.31.0.2#53(172.31.0.2)
  ;; WHEN: Fri Jul 06 16:41:29 CEST 2018
  ;; MSG SIZE rcvd: 133
  ```
### Usefull links
- AWS reference article: [Associating an Amazon VPC and a private hosted zone that you created with different AWS accounts](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zone-private-associate-vpcs-different-accounts.html)
- Terraform support: [Resource: aws_route53_zone_association](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route53_zone_association)
- terraform ticket about support https://github.com/hashicorp/terraform/issues/12465
- tf helper module to automate this thing https://github.com/opetch/terraform-aws-cli-resource