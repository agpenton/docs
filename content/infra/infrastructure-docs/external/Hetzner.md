---
title: "Hetzner"
date: 2022-07-16T16:14:26+02:00
draft: false
---

## About

- `dev.juniqe.com` is the Jenkins master server. It is also used as a sandbox for releases by developers.
- `nexus.juniqe.com` and `aptly.juniqe.com/archive.juniqe.com` are also located on this server.

### dev.juniqe.com

Server has 6TB Hard Drive extension mounted as /data
**Server IP:** 138.201.192.111 is whitelisted by many Security Group in AWS

#### AWS default access

Root user has configured AWS keys created by TF (system-tools/terraform-2.0/providers/aws/global/iam.tf)
With access to the AWS ECR Service for the deployments

### nexus.juniqe.com

Nexus is our repository for java artifacts, which is used during application build.
More details about nexus on official page: https://www.sonatype.com/nexus-repository-oss
The web interface is available at https://nexus.juniqe.com/nexus
Login credentials can be found in SettingsPlugin.scala
Working as a container, initially run with params:

```
#first run
docker run -d -p 8081:8081 --name nexus -v /data/nexus:/sonatype-work sonatype/nexus 

#run nexus after server restart
docker start nexus

#full restart
docker rm -f nexus
docker run -d -p 8081:8081 --name nexus -v /data/nexus:/sonatype-work sonatype/nexus
```

I don't know any other manual configuration for the nexus. Most likely all configuration changes were made via Web Interface
The nginx config for nexus is:

```
server {
  listen 80;
  server_name nexus.juniqe.com;
  return 301 https://$server_name$request_uri;
}


server {
  listen 443 ssl;
  server_name nexus.juniqe.com;
  error_log /var/log/nginx/nexus.error.log;
  access_log /var/log/nginx/nexus.access.log;


  ssl_certificate /home/letsencrypt/juniqe-le.crt;
  ssl_certificate_key /home/letsencrypt/juniqe-le.key;


  client_max_body_size 48M;
  location / {
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Server $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass http://localhost:8081/;
  }

```

### jenkins.juniqe.com

enkins master installation - is not automated. It was a manual setup.
Some confgirations like SAML Auth or Jenkins for Feature Branches are documented.
Besides that Jenkins is regularly update each month to the latest stable version.
For the better security all Credentials for the new builds where moved our from plain text in Jobs to the Credentials Plugin
So each job can get access to the resources without exposing it to users and logs.
More details about Jenkins configuration on this page