---
title: "VPN onboarding"
date: 2022-07-15T13:43:31+02:00
draft: false
---
# Openvpn server

This image is containing all data, including ca cert and server crt/key to run jumphost.
aws s3 bucket juniqe-pki is saving state of easy rsa pki
In order to run this image we need ec2 instance to run docker.
Image for this  instance is build by packer and contains script to update container image on schedule


## Generate new client

* pull latest juniqe-infra
* Authorize in account juniqe as admin
* Authorize in main account docker  container repository
* Get the ca.key pass
* Go to openvpn container definition
    ```
    cd ~/workspace/juniqe-infra/docker/infra/openvpn-server
    ```
* Run client config generation via script with juniqe email as first argument. The script will request password for ca.key
    ```
    # /bin/bash create_client_config.sh  <username>@juniqe.com
    ```
* Get the client conf file from openvpn-data/clients/jumphost02.juniqe.com-<username>@juniqe.com.ovpn


# Revoke

https://github.com/kylemanna/docker-openvpn/blob/1228577d4598762285958ad98724ab37e7b11354/docs/clients.md#revoking-client-certificates

## reconfiguring container
* Read this doc about openvpn docker container https://github.com/kylemanna/docker-openvpn#quick-start

* Synchronize the pki if CA interaction needed
    ```
    bash download_pki_data.sh
    ```

* Change configs in openvpn-data/conf
    * Change ovpn_env.sh to configure server config, as openvpn.conf is generated config
    * Change dynamic_routes to change which dns records will be queried for client route push
* Run docker build
  ```
  docker-compose build
  ```
* Test settings locally
    * Run vpn server
      ```
      docker-compose down -v # ensure old container and volumes are discarded
      docker-compose up 
      ```
    * Setup client config for pointing to 127.0.0.1 as server
    * Run local client against local server

* Publish new image with new settings
    * authorize in ekr
    * docker-compose push
    * Wait 1-2 minutes for updating container on running vpn ec2 server

* Commit changes
* sync PKI changes if they were made
  ```
  bash upload_pki_data.sh
  ```  
## Reconfigure ec2 instance image

* Change configs in jumphost-config
* Run packer
  ```
  packer build packer.json
  ```
## update running instance
* Replace ami id in ~/workspace/juniqe-infra/envs/juniqe-staging/vpn/main.tf
* apply changes
    ```
    cd ~/workspace/juniqe-infra/envs/juniqe-staging/vpn
    terraform apply
    ```
* commit changes    
