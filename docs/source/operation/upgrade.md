# Upgrading to v4

## Overview

As v4 introduces non-backward compatibility in terms of data structure, this guide outlines the steps required to complete the upgrade from Gluu Server v3.

## Exporting Data

1.  Make sure to backup existing LDAP data

1.  Set environment variable as a placeholder for LDAP server password (for later use):

    ```sh
    export LDAP_PASSWD=YOUR_PASSWORD_HERE
    ```

1.  Assuming that existing LDAP container called `ldap` has data, export data from each backend:

    1.  Export `o=gluu`

        ```sh
        docker exec -ti ldap /opt/opendj/bin/ldapsearch \
            -Z \
            -X \
            -D "cn=directory manager" \
            -w $LDAP_PASSWD \
            -p 1636 \
            -b "o=gluu" \
            -s sub \
            'objectClass=*' > gluu.ldif
        ```

    1.  Export `o=site`

        ```sh
        docker exec -ti ldap /opt/opendj/bin/ldapsearch \
            -Z \
            -X \
            -D "cn=directory manager" \
            -w $LDAP_PASSWD \
            -p 1636 \
            -b "o=site" \
            -s sub \
            'objectClass=*' > site.ldif
        ```


    1.  Export `o=metric`

        ```sh
        docker exec -ti ldap /opt/opendj/bin/ldapsearch \
            -Z \
            -X \
            -D "cn=directory manager" \
            -w $LDAP_PASSWD \
            -p 1636 \
            -b "o=metric" \
            -s sub \
            'objectClass=*' > metric.ldif
        ```

1.  Unset `LDAP_PASSWD` environment variable

## Migrating Data

1.  Deploy a temporary Wren:DS container; here's an example for Docker Compose:

    ```yaml
    services:
      tmp_ldap:
        image: gluufederation/wrends:4.0.1_02
        environment:
          - GLUU_CONFIG_CONSUL_HOST=consul
          - GLUU_SECRET_VAULT_HOST=vault
          - GLUU_CERT_ALT_NAME=ldap
          - GLUU_LDAP_AUTO_REPLICATE=false
        container_name: tmp_ldap
        volumes:
          - ./volumes/v4/opendj/config:/opt/opendj/config
          - ./volumes/v4/opendj/ldif:/opt/opendj/ldif
          - ./volumes/v4/opendj/logs:/opt/opendj/logs
          - ./volumes/v4/opendj/db:/opt/opendj/db
          - ./volumes/v4/opendj/flag:/flag
          - ./volumes/v4/opendj/backup:/opt/opendj/bak
          - ./vault_role_id.txt:/etc/certs/vault_role_id
          - ./vault_secret_id.txt:/etc/certs/vault_secret_id
        restart: unless-stopped
        labels:
          - "SERVICE_IGNORE=yes"
    ```

    !!! Note
        **DO NOT** mount existing volumes used by `ldap` container; instead mount different volumes into `tmp_ldap` container.

    Afterwards run the container using the following command

    ```sh
    docker-compose up -d tmp_ldap
    ```

    and wait until container running completely.

1.  Run `gluufederation/upgrade:4.0.1_01` to migrate existing LDAP data

    ```sh
    docker run \
        --rm \
        --net container:consul \
        -e GLUU_CONFIG_CONSUL_HOST=consul \
        -e GLUU_SECRET_VAULT_HOST=vault \
        -e GLUU_LDAP_URL=tmp_ldap:1636 \
        -v $PWD/vault_role_id.txt:/etc/certs/vault_role_id \
        -v $PWD/vault_secret_id.txt:/etc/certs/vault_secret_id \
        -v $PWD/scripts:/app/scripts \
        -v $PWD/gluu.ldif:/app/imports/gluu.ldif \
        -v $PWD/site.ldif:/app/imports/site.ldif \
        -v $PWD/metric.ldif:/app/imports/metric.ldif \
        gluufederation/upgrade:4.0.1_01 --source 3.1.6 --target 4.0
    ```

    The migration process may take sometime; please wait until it is completed.

    !!! Note
        The upgrade process doesn't update custom scripts for oxAuth/oxTrust to avoid overwriting scripts that were modified by users. They must be updated them manually.

3.  Once migration process is completed, remove the `tmp_ldap` container:

    ```sh
    docker rm -f tmp_ldap
    ```

## Switching to v4 Containers

Once LDAP data already migrated, upgrade the container version to v4. Here's an example of Docker Compose manifest for v4:

```yaml
# use v2.x API to allow `mem_limit` option
version: "2.4"

services:
  consul:
    image: consul
    command: agent -server -bootstrap -ui
    hostname: consul-1
    environment:
      - CONSUL_BIND_INTERFACE=eth0
      - CONSUL_CLIENT_INTERFACE=eth0
    container_name: consul
    restart: unless-stopped
    volumes:
      - ./volumes/consul:/consul/data
    labels:
      - "SERVICE_IGNORE=yes"
    restart: unless-stopped

  vault:
    container_name: vault
    image: vault:1.0.1
    command: vault server -config=/vault/config
    volumes:
      - ./volumes/vault/config:/vault/config
      - ./volumes/vault/data:/vault/data
      - ./volumes/vault/logs:/vault/logs
      - ./vault_gluu_policy.hcl:/vault/config/policy.hcl
    cap_add:
      - IPC_LOCK
    environment:
      - VAULT_REDIRECT_INTERFACE=eth0
      - VAULT_CLUSTER_INTERFACE=eth0
      - VAULT_ADDR=http://0.0.0.0:8200
      - VAULT_LOCAL_CONFIG={"backend":{"consul":{"address":"consul:8500","path":"vault/"}},"listener":{"tcp":{"address":"0.0.0.0:8200","tls_disable":1}}}
    restart: unless-stopped
    depends_on:
      - consul
    labels:
      - "SERVICE_IGNORE=yes"

  registrator:
    image: gliderlabs/registrator:master
    command: registrator -internal -cleanup -resync 30 -retry-attempts 5 -retry-interval 10 consul://consul:8500
    container_name: registrator
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock
    restart: unless-stopped
    depends_on:
      - consul

  nginx:
    image: gluufederation/nginx:4.0.1_02
    environment:
      - GLUU_CONFIG_CONSUL_HOST=consul
      - GLUU_SECRET_VAULT_HOST=vault
    ports:
      - "80:80"
      - "443:443"
    container_name: nginx
    restart: unless-stopped
    labels:
      - "SERVICE_IGNORE=yes"
    volumes:
      - ./vault_role_id.txt:/etc/certs/vault_role_id
      - ./vault_secret_id.txt:/etc/certs/vault_secret_id

  ldap:
    image: gluufederation/wrends:4.0.1_02
    environment:
      - GLUU_CONFIG_CONSUL_HOST=consul
      - GLUU_SECRET_VAULT_HOST=vault
      # the value must match service name `ldap` because other containers
      # use this value as LDAP hostname
      - GLUU_CERT_ALT_NAME=ldap
      - GLUU_LDAP_ADVERTISE_ADDR=ldap
      - GLUU_PERSISTENCE_TYPE=ldap
      - GLUU_PERSISTENCE_LDAP_MAPPING=default
    container_name: ldap
    volumes:
      - ./volumes/v4/opendj/config:/opt/opendj/config
      - ./volumes/v4/opendj/ldif:/opt/opendj/ldif
      - ./volumes/v4/opendj/logs:/opt/opendj/logs
      - ./volumes/v4/opendj/db:/opt/opendj/db
      - ./volumes/v4/opendj/backup:/opt/opendj/bak
      - ./vault_role_id.txt:/etc/certs/vault_role_id
      - ./vault_secret_id.txt:/etc/certs/vault_secret_id
    restart: unless-stopped
    labels:
      - "SERVICE_IGNORE=yes"
```

!!! Note
    Remember to mount correct volumes for `ldap` service. In example above, `volumes/v4/opendj` contains migrated data from v3.
