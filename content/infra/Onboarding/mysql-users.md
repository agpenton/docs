---
title: "MySQL users onboarding"
date: 2022-07-15T13:43:31+02:00
draft: false
---
## Read access to production mysql database

- [this is outdated, the new process is different] The form should be filled https://goo.gl/forms/lC5JMNm3GsogmOlW2
- Provide access via connecting to master and run commands; Choose a strong password.
    ```
    mysql -vv -h masterdb.juniqe-production.  -u master -p
    mysql> CREATE USER 'ro_firstname_lastname'@'172.31.13.38' IDENTIFIED BY 'CHANGE_TO_SECURE_PASSWORD';
    mysql> GRANT select,usage,show view on juniqe.* TO 'ro_firstname_lastname'@'172.31.13.38' ;
    ```
- simple script to automate account creation Expand source:

    ```
    #!/bin/bash
     
    set -ue -o pipefail
     
    function q() {
        mysql -vv -h masterdb.juniqe-production  -D juniqe -u master -p -e "${1}"
    }
     
    firstname="${1}"
    lastname="${2}"
    # email="${3}"
    full_username="ro_${firstname}_${lastname:0:1}"
    username=${full_username:0:16}
    allowed_ip="172.31.13.38"
    # allowed_ip="10.0.101.179"
     
    pwd_data="$(curl -s -q -d 'secret=SECRET&ttl=604800'  -u 'marat@juniqe.com:<api token>' https://onetimesecret.com/api/v1/generate)"
     
     
    password="$(echo $pwd_data | jq -r  ".value" )"
    password_link="https://onetimesecret.com/secret/$(echo $pwd_data | jq -r  ".secret_key")"
     
    echo "${username}" | head -c 16 | wc
     
    q "
    BEGIN;
    CREATE USER  '${username}'@'${allowed_ip}' IDENTIFIED BY '${password}';
    GRANT select,usage,show view on juniqe.* TO ${username}@${allowed_ip};
    COMMIT;
    "
     
    mysql -vv -h masterdb.juniqe-production  -D juniqe -u"${username}" -p${password} -e "select 1;"
    message="
     
    This is your access to juniqe database
    Host: "slavedb.juniqe.com"
    Port: 3306
    Username: ${username}
    Password: Open this link to get it ${password_link} (This link will expire after 7 days)
     
    VPN should be activated before mysql connection
    "
     
    echo "${message}"
    ```
- Provide connection information to user via secure channel.

```
Provide connection information to user via secure channel.

Host: slavedb.juniqe-production

Username: ro_firstname_lastname

Password: 'CHANGE_TO_SECURE_PASSWORD'

Database: juniqe

Port: 3306

VPN should be activated in order to use it
```