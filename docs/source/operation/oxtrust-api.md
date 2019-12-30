## Overview

The following sections are guides on how to access oxTrust API using within Gluu Server container deployment.
See [oxTrust API docs](https://gluu.org/docs/oxtrust-api/4.0/) for reference.

## Prerequisites

1. `gluufederation/config-init:4.0.1_05` image (test mode client is introduced).
1. `gluufederation/persistence:4.0.1_05` image (enable oxTrust API upon deployment).
1. `gluufederation/oxauth:4.0.1_05` image.
1. `gluufederation/oxtrust:4.0.1_05` image.

## Available API Modes

The oxTrust API has two modes that administrators can configure according to need.

### Test Mode

!!! Note
    Test mode is not recommended for production. Choose UMA mode instead.

1.  Set environment variable `GLUU_OXTRUST_API_ENABLED=true` and `GUU_OXTRUST_API_TEST_MODE=true` when running `gluufederation/persistence` container to enable oxTrust API:

    ```sh
    docker run \
        --rm \
        --name persistence \
        -e GLUU_CONFIG_CONSUL_HOST=consul \
        -e GLUU_SECRET_VAULT_HOST=vault \
        -e GLUU_PERSISTENCE_TYPE=ldap \
        -e GLUU_PERSISTENCE_LDAP_MAPPING=default \
        -e GLUU_LDAP_URL=ldap:1636 \
        -e GLUU_OXTRUST_API_ENABLED=true \
        -e GLUU_OXTRUST_API_TEST_MODE=true \
        -v $PWD/vault_role_id.txt:/etc/certs/vault_role_id \
        -v $PWD/vault_secret_id.txt:/etc/certs/vault_secret_id \
        gluufederation/persistence:4.0.1_05
    ```
    
    If using kubernetes `create.sh` answer yes to both the following prompts: 
    
    ```sh
    Enable oxTrust Api         [N]?[Y/N]                               y
    Enable oxTrust Test Mode [N]?[Y/N]                                 y
    ```
    
    Alternatively, enable the features using oxTrust UI. 

1.  Obtain Test mode client credentials from config and secret backends.

    1.  Grab `api_test_client_id` from config backend; in this example, we're getting `0008-b52a8524-35b2-4835-968e-481a366be8cd` as its value. This is the client ID.

    1.  Grab `api_test_client_secret` from secret backend; in this example, we're getting `TVtZwLZxp25XFDelMJNDQsa8` as its value. This is the client secret.

1.  Get token from Gluu Server; in this example we're using `https://demoexample.gluu.org`.

    ```sh
    curl -k -u '0008-b52a8524-35b2-4835-968e-481a366be8cd:TVtZwLZxp25XFDelMJNDQsa8' \
        https://demoexample.gluu.org/oxauth/restv1/token \
        -d grant_type=client_credentials
    ```

    The response example:

    ```json
    {
        "access_token": "0d14102c-70e5-485c-8b64-e56f1ecfcf3e",
        "token_type": "bearer",
        "expires_in": 299
    }
    ```

    Extract the `access_token` value (in this case, `0d14102c-70e5-485c-8b64-e56f1ecfcf3e` is the token).

1.  Make request to oxTrust API endpoints:

    ```sh
    curl -k -H 'Authorization: Bearer 0d14102c-70e5-485c-8b64-e56f1ecfcf3e' \
        https://demoexample.gluu.org/identity/restv1/api/v1/groups
    ```

    If succeed, the output is similar to the following:

    ```json
    [
        {
            "status": "ACTIVE",
            "displayName": "Gluu Manager Group",
            "description": "This group is for administrative purpose, with full acces to users",
            "members": [
                "inum=04f0d0e9-a609-4a0a-9580-ebd981a49a61,ou=people,o=gluu"
            ],
            "inum": "60B7",
            "owner": null,
            "organization": null,
            "iname": null
        }
    ]
    ```

### UMA Mode

1.  Set environment variable `GLUU_OXTRUST_API_ENABLED=true` when running `gluufederation/persistence` container to enable oxTrust API:

    ```sh
    docker run \
        --rm \
        --name persistence \
        -e GLUU_CONFIG_CONSUL_HOST=consul \
        -e GLUU_SECRET_VAULT_HOST=vault \
        -e GLUU_PERSISTENCE_TYPE=ldap \
        -e GLUU_PERSISTENCE_LDAP_MAPPING=default \
        -e GLUU_LDAP_URL=ldap:1636 \
        -e GLUU_OXTRUST_API_ENABLED=true \
        -e GLUU_OXTRUST_API_TEST_MODE=false \
        -v $PWD/vault_role_id.txt:/etc/certs/vault_role_id \
        -v $PWD/vault_secret_id.txt:/etc/certs/vault_secret_id \
        gluufederation/persistence:4.0.1_05
    ```
    
    If using kubernetes `create.sh` answer `Y` to enabling `oxTrust API` and `N` to enabling `Test Mode`.
    
    ```sh
    Enable oxTrust Api         [N]?[Y/N]                               y
    Enable oxTrust Test Mode [N]?[Y/N]                                 N
    ```
    Alternatively, enable the features using oxTrust UI.

1.  Make request to oxTrust API (in this example, we're going to use `https://demoexample.gluu.org` URL), for example:

    ```sh
    curl -k -I https://demoexample.gluu.org/identity/restv1/api/v1/groups
    ```

    The request is rejected due to unauthenticated client and the response headers will be similar as the following:

    ```
    HTTP/1.1 401 Unauthorized
    WWW-Authenticate: UMA realm="Authorization required", host_id=demoexample.gluu.org, as_uri=https://demoexample.gluu.org/.well-known/uma2-configuration, ticket=ed5d9fa7-7117-4fc0-85c2-17a064448dc8
    ```

    Extract the ticket from `WWW-Authenticate` header; in this example the ticket is `ed5d9fa7-7117-4fc0-85c2-17a064448dc8`.

1.  Copy `api-rp.jks` and `api-rp-keys.json` from oxAuth container into host:

    ```sh
    docker cp oxauth:/etc/certs/api-rp.jks api-rp.jks \
        && docker cp oxauth:/etc/certs/api-rp-keys.json api-rp-keys.json
    ```
    
    In kubernetes get the oxauth pod name and use the following commands:
  
      ```sh
    kubectl cp oxauth-acsacsd2123:etc/certs/api-rp.jks api-rp.jks \
        && kubectl cp oxauth-acsacsd2123:etc/certs/api-rp-keys.json api-rp-keys.json
    ```
    


1.  Determine algorithm for signing JWT string, i.e. `RS256`.

    Here's an example of `api-rp-keys.json` contents:

    ```json
    {
        "keys": [
            {
                "kty": "RSA",
                "e": "AQAB",
                "use": "sig",
                "crv": "",
                "kid": "777c0619-802c-480d-9432-a5e25f85867a_sig_rs256",
                "x5c": ["MIIDAzCCAeugAwIBAgIgIoDkhKXYZG5/LDPoUEUBxLpvsUDwL+OEzkAMuMpglzLH6g9dDUyGVEh8iRg=="],
                "exp": 1606219942910,
                "alg": "RS256",
                "n": "r3LItabzy3Lg0SXf_6EZ1oANjyYQ_HCEj-r5cynyD7dnAQdXvkRLVMAby0EAoCaeEo_QkU79BCOY6o2w"
            }
        ]
    }
    ```

    Make sure `alg` value is `RS256`.
    Grab the `kid` value (in this example `777c0619-802c-480d-9432-a5e25f85867a_sig_rs256`).

1.  Grab keystore password from secret backend where key is `api_rp_client_jks_pass`. In this example, we're getting `secret` as its value.

1.  Convert `api-rp.jks` to `api-rp.pkcs12` (delete existing `api-rp.pkcs12` file if any):

    ```sh
    keytool -importkeystore \
        -srckeystore api-rp.jks \
        -srcstorepass secret  \
        -srckeypass secret \
        -srcalias 777c0619-802c-480d-9432-a5e25f85867a_sig_rs256 \
        -destalias 777c0619-802c-480d-9432-a5e25f85867a_sig_rs256 \
        -destkeystore api-rp.pkcs12 \
        -deststoretype PKCS12 \
        -deststorepass secret \
        -destkeypass secret
    ```

1.  Extract public and private key pair from `api-rp.pkcs12`:

    ```sh
    openssl pkcs12 -in api-rp.pkcs12 -nodes -out api-rp.pem -passin pass:secret
    ```

    Here's an example of generated `api-rp.pem`:

    ```text
    Bag Attributes
        friendlyName: d405b162-fb5d-4e9f-81cd-4edcb0db486a_sig_rs256
        localKeyID: 54 69 6D 65 20 31 35 37 35 31 37 34 30 33 35 32 33 30
    Key Attributes: <No Attributes>
    -----BEGIN PRIVATE KEY-----
    MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQCwvKNUOZxaOcRB
    wwFtCJIqoqnaPYA0kfJEnnnm
    -----END PRIVATE KEY-----
    Bag Attributes
        friendlyName: d405b162-fb5d-4e9f-81cd-4edcb0db486a_sig_rs256
        localKeyID: 54 69 6D 65 20 31 35 37 35 31 37 34 30 33 35 32 33 30
    subject=/CN=oxAuth CA Certificates
    issuer=/CN=oxAuth CA Certificates
    -----BEGIN CERTIFICATE-----
    MIIDAzCCAeugAwIBAgIgAPS1X/1F5GFLp8xNYLbw9zs34TOwSd2Kz++dZHRijIkw
    grfXl0CuwA==
    -----END CERTIFICATE-----
    ```

    Grab the string starts with `-----BEGIN PRIVATE KEY-----` and ends with `-----END PRIVATE KEY-----`.
    This is the private key.

1.  Prepare data for generating JWT string.

    1.  Header

        ```json
        {
            "typ": "JWT",
            "alg": "RS256",
            "kid": "777c0619-802c-480d-9432-a5e25f85867a_sig_rs256"
        }
        ```

    1.  Payload

        Grab client ID from config backend where key is `oxtrust_requesting_party_client_id`. In this example we're getting `0008-76f0b100-6d68-4f21-96ca-c6e49d30094b` as its value.

        ```json
        {
            "iss": "0008-76f0b100-6d68-4f21-96ca-c6e49d30094b",
            "sub": "0008-76f0b100-6d68-4f21-96ca-c6e49d30094b",
            "exp": 1575185573,
            "iat": 1575181565,
            "jti": "2f1c50c6-0359-4913-b3f9-c17ca93a1b82",
            "aud": "https://demoexample.gluu.org/oxauth/restv1/token"
        }
        ```

        Note:

        - `iat` value is time since epoch; we can use `date +%s` command to get a value
        - `exp` value is expiration since epoch; we can use `date --date="1 hours" +%s`
        - `jti` value must be unique; we can use `uuidgen` command to get a value
        - `aud` is the URL for getting the token
        - `iss` and `sub` are client ID

    1.  Private key (see previous section about extracting private key)

    After header, payload, and private key are ready, generate JWT string using [debugger](https://jwt.io/#debugger-io) or any of supported [libraries](https://jwt.io/#libraries-io). Save the JWT string, for example:

    ```text
    eyJhbGciOiJSUzI1NiIs.RiLZyW2yYdF4P0QD0oY9zjBfsFwFSpSCRUe.3WnaETMtAIPpXQhry6SYFR1tFv1t4XO14o1qVA
    ```

1.  Grab token from `https://demoexample.gluu.org/oxauth/restv1/token`:

    ```sh
    curl -k https://demoexample.gluu.org/oxauth/restv1/token \
        -d grant_type='urn:ietf:params:oauth:grant-type:uma-ticket' \
        -d ticket='ed5d9fa7-7117-4fc0-85c2-17a064448dc8'  \
        -d client_id='0008-76f0b100-6d68-4f21-96ca-c6e49d30094b' \
        -d client_assertion_type='urn:ietf:params:oauth:client-assertion-type:jwt-bearer' \
        -d client_assertion='eyJhbGciOiJSUzI1NiIs.RiLZyW2yYdF4P0QD0oY9zjBfsFwFSpSCRUe.3WnaETMtAIPpXQhry6SYFR1tFv1t4XO14o1qVA'
    ```

    The response example:

    ```json
    {
        "access_token": "d01cdc70-6519-4118-89bb-9ee1748acdd1_D8F4.E104.D094.6B62.79E0.6F8E.FB55.0509",
        "token_type": "Bearer",
        "upgraded": false,
        "pct": "0b598193-11cb-4926-9e4c-cc7c95e6ce37_CB14.8C6B.E2B2.46CD.53B2.CBEF.C50E.9FE9"
    }
    ```

    Extract value of `access_token`; in this case `d01cdc70-6519-4118-89bb-9ee1748acdd1_D8F4.E104.D094.6B62.79E0.6F8E.FB55.0509`.

1.  Retry request to get groups (this time pass along the token in the request header):

    ```sh
    curl -k -H 'Authorization: Bearer d01cdc70-6519-4118-89bb-9ee1748acdd1_D8F4.E104.D094.6B62.79E0.6F8E.FB55.0509' \
        https://demoexample.gluu.org/identity/restv1/api/v1/groups
    ```

    If succeed, the output is similar to the following:

    ```json
    [
        {
            "status": "ACTIVE",
            "displayName": "Gluu Manager Group",
            "description": "This group is for administrative purpose, with full acces to users",
            "members": [
                "inum=04f0d0e9-a609-4a0a-9580-ebd981a49a61,ou=people,o=gluu"
            ],
            "inum": "60B7",
            "owner": null,
            "organization": null,
            "iname": null
        }
    ]
    ```

    Reuse the token (as long as it still valid) to make any request to oxTrust API endpoints.
