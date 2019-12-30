## Overview

Docker image packaging for OpenDJ/Wren:DS.

## Versions

- Stable: `gluufederation/wrends:4.0.1_02`.
- Unstable: `gluufederation/wrends:4.0.1_dev`.

Refer to [Changelog](https://github.com/GluuFederation/docker-opendj/blob/4.0/CHANGES.md) for details on new features, bug fixes, or older releases.

!!!Note
    Starting from v4, `gluufederation/wrends` is a drop-in replacement for older `gluufederation/opendj` image. For older versions, stick with `gluufederation/opendj` instead.

## Environment Variables

The following environment variables are supported by the container:

- `GLUU_CONFIG_ADAPTER`: The config backend adapter, can be `consul` (default) or `kubernetes`.
- `GLUU_CONFIG_CONSUL_HOST`: hostname or IP of Consul (default to `localhost`).
- `GLUU_CONFIG_CONSUL_PORT`: port of Consul (default to `8500`).
- `GLUU_CONFIG_CONSUL_CONSISTENCY`: Consul consistency mode (choose one of `default`, `consistent`, or `stale`). Default to `stale` mode.
- `GLUU_CONFIG_CONSUL_SCHEME`: supported Consul scheme (`http` or `https`).
- `GLUU_CONFIG_CONSUL_VERIFY`: whether to verify cert or not (default to `false`).
- `GLUU_CONFIG_CONSUL_CACERT_FILE`: path to Consul CA cert file (default to `/etc/certs/consul_ca.crt`). This file will be used if it exists and `GLUU_CONFIG_CONSUL_VERIFY` set to `true`.
- `GLUU_CONFIG_CONSUL_CERT_FILE`: path to Consul cert file (default to `/etc/certs/consul_client.crt`).
- `GLUU_CONFIG_CONSUL_KEY_FILE`: path to Consul key file (default to `/etc/certs/consul_client.key`).
- `GLUU_CONFIG_CONSUL_TOKEN_FILE`: path to file contains ACL token (default to `/etc/certs/consul_token`).
- `GLUU_CONFIG_KUBERNETES_NAMESPACE`: Kubernetes namespace (default to `default`).
- `GLUU_CONFIG_KUBERNETES_CONFIGMAP`: Kubernetes configmaps name (default to `gluu`).
- `GLUU_CONFIG_KUBERNETES_USE_KUBE_CONFIG`: Load credentials from `$HOME/.kube/config`, only useful for non-container environment (default to `false`).
- `GLUU_SECRET_ADAPTER`: The secrets adapter, can be `vault` or `kubernetes`.
- `GLUU_SECRET_VAULT_SCHEME`: supported Vault scheme (`http` or `https`).
- `GLUU_SECRET_VAULT_HOST`: hostname or IP of Vault (default to `localhost`).
- `GLUU_SECRET_VAULT_PORT`: port of Vault (default to `8200`).
- `GLUU_SECRET_VAULT_VERIFY`: whether to verify cert or not (default to `false`).
- `GLUU_SECRET_VAULT_ROLE_ID_FILE`: path to file contains Vault AppRole role ID (default to `/etc/certs/vault_role_id`).
- `GLUU_SECRET_VAULT_SECRET_ID_FILE`: path to file contains Vault AppRole secret ID (default to `/etc/certs/vault_secret_id`).
- `GLUU_SECRET_VAULT_CERT_FILE`: path to Vault cert file (default to `/etc/certs/vault_client.crt`).
- `GLUU_SECRET_VAULT_KEY_FILE`: path to Vault key file (default to `/etc/certs/vault_client.key`).
- `GLUU_SECRET_VAULT_CACERT_FILE`: path to Vault CA cert file (default to `/etc/certs/vault_ca.crt`). This file will be used if it exists and `GLUU_SECRET_VAULT_VERIFY` set to `true`.
- `GLUU_SECRET_KUBERNETES_NAMESPACE`: Kubernetes namespace (default to `default`).
- `GLUU_SECRET_KUBERNETES_CONFIGMAP`: Kubernetes secrets name (default to `gluu`).
- `GLUU_SECRET_KUBERNETES_USE_KUBE_CONFIG`: Load credentials from `$HOME/.kube/config`, only useful for non-container environment (default to `false`).
- `GLUU_WAIT_MAX_TIME`: How long the startup "health checks" should run (default to `300` seconds).
- `GLUU_WAIT_SLEEP_DURATION`: Delay between startup "health checks" (default to `10` seconds).
- `GLUU_LDAP_ADDR_INTERFACE`: interface name where the IP will be guessed and registered as OpenDJ host, e.g. eth0 (will be ignored if `GLUU_LDAP_ADVERTISE_ADDR` is used).
- `GLUU_LDAP_ADVERTISE_ADDR`: the hostname/IP address used as the host of OpenDJ server.
- `GLUU_LDAP_AUTO_REPLICATE`: enable replication automatically (default to `true`).
- `GLUU_CERT_ALT_NAME`: an additional DNS name set as Subject Alt Name in cert. If the value is not an empty string and doesn't match existing Subject Alt Name (or doesn't exist) in existing cert, then new cert will be regenerated and overwrite the one that saved in config backend. This environment variable is __required only if__ oxShibboleth is deployed, to address issue with mismatched `CN` and destination hostname while trying to connect to OpenDJ. Note, any existing containers that connect to OpenDJ must be re-deployed to download new cert.

## Initializing LDAP Data

Starting from v4, the container will not import the initial data into LDAP. Use [Persistence](../persistence/#initializing-data) container to do so.

## LDAP Replication

The replication process is automatically run when the container runs, only if these conditions are met:

1. There are multiple LDAP containers running in the cluster
2. The `o=gluu`, `o=site`, `o=metric` backends have not been replicated nor have entries

Check the LDAP container logs to see the result and optionally run `/opt/opendj/bin/dsreplication status` inside the container.
