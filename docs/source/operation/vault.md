## Overview

This operational guide is for a Gluu Server deployment that uses `vault` as the `GLUU_SECRET_ADAPTER` backend.
If using a Kubernetes deployment, this guide is optional.

## Choosing a Storage Backend

The storage backend determines where to store secrets. For ease of use and HA mode, we recommend the `consul` backend.

Example:

```sh
docker run \
    -e 'VAULT_LOCAL_CONFIG={"backend":{"consul":{"address":"consul.server:8500","path":"vault/"}},"listener":{"tcp":{"address":"0.0.0.0:8200","tls_disable":1}}}' \
    vault:1.0.1
```

See [Vault storage backends](https://www.vaultproject.io/docs/configuration/storage/index.html) for details.

## Adding Custom Policy

To distinguish secrets used by Gluu Server and others, we need to apply a custom policy.
Create the following policy and save it to a file (in this example, we will save it as `vault_gluu_policy.hcl`):

```
path "secret/gluu/*" {
    capabilities = ["create", "list", "read", "delete", "update"]
}
```

There are various ways to _mount_ this file into Vault container:

-   Mount `vault_gluu_policy.hcl` volume directly from host's filesystem:

    ```sh
    docker run \
        -v /path/to/vault_gluu_policy.hcl:/vault/config/policy.hcl \
        vault:1.0.1
    ```

-   Save the file to `docker config` and inject it into container (only available in Docker Swarm Mode):

    ```sh
    docker config create vault_gluu_policy vault_gluu_policy.hcl
    docker service create \
        --name vault \
        --config src=vault_gluu_policy,target=/vault/config/policy.hcl \
        vault:1.0.1
    ```

## Strategies for Unsealing Vault

Vault uses [seal and unseal](https://www.vaultproject.io/docs/concepts/seal.html) to lock/unlock the secrets.
When a Vault server starts, it remains ina  sealed state until the unseal process is successfully executed.
Note that when the Vault server is stopped or restarted, it will be sealed again, so we need to determine strategies on how to unseal Vault upfront before deploying Vault container.

There are two ways to unseal Vault:

-   __manual__ process which involves human intervention to run specific command for each Vault container and enters one or more keys.
-   __auto-unseal__ which requires [3rd-party services](https://www.vaultproject.io/docs/configuration/seal/index.html).

For ease of use, we recommend using auto-unseal to reduce the complexity of unsealing Vault.
One of the services we recommend is GCP KMS.

Subscribe to the GCP KMS service and obtain the credentials file, similar to the following contents:

```json
{
    "type": "service_account",
    "project_id": "<project-name>",
    "private_key_id": "<private-key-id>",
    "private_key": "<private-key>",
    "client_email": "<service-name>@<project-name>.iam.gserviceaccount.com",
    "client_id": "<client-id>",
    "auth_uri": "https://accounts.google.com/o/oauth2/auth",
    "token_uri": "https://oauth2.googleapis.com/token",
    "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
    "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/<service-name>%40<project-name>.iam.gserviceaccount.com"
}
```

Replace the value enclosed in `<>` characters with appropriate entries, then save it to a file `gcp_kms_creds.json`.

Create seal configuration (stanza), adjust the value enclosed in `<>` characters, then save it as `gcp_kms_stanza.hcl`.

```
seal "gcpckms" {
    credentials = "/vault/config/creds.json"
    project     = "<project-name>"
    region      = "<region-name>"
    key_ring    = "<key-ring>"
    crypto_key  = "<crypto-key>"
}
```

Those two files above need to be _mounted_ into Vault container, either directly or using `docker secret`.

 -   Mount `gcp_kms_creds.json` and `gcp_kms_stanza.hcl` volumes directly from host's filesystem:

    ```sh
    docker run \
        -v /path/to/gcp_kms_creds.json:/vault/config/creds.json \
        -v /path/to/gcp_kms_stanza.hcl:/vault/config/stanza.hcl \
        vault:1.0.1
    ```

-   Save the files to `docker secret` and inject them into the container (only available in Docker Swarm Mode):

    ```sh
    docker secret create gcp_kms_creds gcp_kms_creds.json
    docker secret create gcp_kms_stanza gcp_kms_stanza.hcl
    docker service create \
        --name vault \
        --secret src=gcp_kms_creds,target=/vault/config/creds.json \
        --secret src=gcp_kms_stanza,target=/vault/config/stanza.hcl \
        vault:1.0.1
    ```

## Basic Vault Deployment

Given that the custom policy is already created, as well as GCP KMS credentials and stanza, we may deploy Vault container with the following command:

```sh
docker run \
    --cap-add=IPC_LOCK \
    --name vault \
    -e VAULT_CLUSTER_INTERFACE=eth0 \
    -e VAULT_REDIRECT_INTERFACE=eth0 \
    -e VAULT_ADDR=http://0.0.0.0:8200 \
    -e 'VAULT_LOCAL_CONFIG={"backend":{"consul":{"address":"consul.server:8500","path":"vault/"}},"listener":{"tcp":{"address":"0.0.0.0:8200","tls_disable":1}}}' \
    -e /path/to/vault_gluu_policy.hcl:/vault/config/policy.hcl \
    -e /path/to/gcp_kms_creds.json:/vault/config/creds.json \
    -e /path/to/gcp_kms_stanza.hcl:/vault/config/stanza.hcl \
    vault:1.0.1 vault server -config=/vault/config
```

## Initializing Vault

Before using Vault secrets, we need to initialize it. Note that this process is only executed once, so we can check whether Vault has been initialized or not:

```sh
docker exec vault vault operator init -status
```

If the output returns `Vault is not initialized`, then we need to initialize it first.

```sh
docker exec vault vault operator init \
    -key-shares=1 \
    -key-threshold=1 \
    -recovery-shares=1 \
    -recovery-threshold=1
```

The command above returns output similar to the following contents:

```
Unseal Key 1: <random-key>

Initial Root Token: <random-token>

Vault initialized with 1 key shares and a key threshold of 1. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 1 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 1 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault rekey" for more information.
```

Note the `<random-key>` and `<random-token>` values as seen above for later use.

Make sure that Vault has been initialized:

```sh
docker exec vault vault operator init -status
```

If the output is `Vault is initialized`, then we initialized Vault properly.

## Log in to Vault

Some commands require authenticated login.
To log in to Vault, run the command below:

```sh
docker exec -ti vault vault login -no-print
```

When prompted for a token, enter the `<random-token>` value that you noted in the last step.

## Enabling Secret Engine

Gluu Server containers only support Vault's [KV v1](https://www.vaultproject.io/docs/secrets/kv/kv-v1.html).

Starting from Vault 1.1.0, secret engine must be enabled manually, for example:

```sh
docker run vault vault secrets enable -version=1 -path=secret kv
```

## Enabling Custom Policy

As we have _mounted_ a custom policy file, we need to enable it:

```sh
docker exec vault vault policy write gluu /vault/config/policy.hcl
```

## Enabling AppRole Auth

[AppRole](https://www.vaultproject.io/docs/auth/approle.html) auth is used by Gluu Server containers to access Vault secrets, so we need to enable and map the policy onto it:

```sh
docker exec vault vault auth enable approle
docker exec vault vault write auth/approle/role/gluu policies=gluu
docker exec vault vault write auth/approle/role/gluu \
    secret_id_ttl=0 \
    token_num_uses=0 \
    token_ttl=20m \
    token_max_ttl=30m \
    secret_id_num_uses=0
```

Afterwards we need to generate AppRole `RoleID` and `SecretID` which similar to username and password combination, but for machines/apps.

For RoleID, execute the following command:

```sh
docker exec vault vault read -field=role_id auth/approle/role/gluu/role-id
```

Save the output as `vault_role_id.txt` to a file (and `docker secret` or Kubernetes `secrets` if needed).

For SecretID, execute the following command:

```sh
docker exec vault vault write -f -field=secret_id auth/approle/role/gluu/secret-id
```

Save the output as `vault_secret_id.txt` to a file (and `docker secret` or Kubernetes `secrets` if needed).

!!! Note
    The RoleID and SecretID values are required by all Gluu Server containers.

## Unsealing Vault Manually

As mentioned earlier, Vault can be unsealed either manually or automatically. The latter is relatively easy to achieve and doesn't require human intervention during the unseal process, whereas the former requires human intervention or script to automate the unseal process whenever Vault server is restarted.

To check whether Vault has been unsealed, run the following command:

```sh
docker exec vault vault status
```

which will return this output:

```
Key                Value
---                -----
Seal Type          shamir
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    0/3
Unseal Nonce       n/a
Version            1.0.1
HA Enabled         true
```

If `Sealed` value is `true`, run the command below:

```sh
docker exec -ti vault vault unseal
```

When prompted, enter the `<random-key>` from the [Initializing Vault](#initializing-vault) section.

Re-run the `docker exec vault vault status` and check the `Sealed` value.
Repeat manual unseal until `Sealed` value is `false`.
