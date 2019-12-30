## Overview

The role of KeyRotation container is to regenerate `oxauth-keys.jks`. After keys have been regenerated, these keys will be saved into the secrets backend. On the other hand, the oxAuth container runs periodic task to pull new `oxauth-keys.jks` (if any) from secrets backend.

### Deploying the container

Below is an example of `docker-compose.yml` to deploy the KeyRotation container:

```yaml
services:
  key-rotation:
    image: gluufederation/key-rotation:4.0.1_05
    environment:
      - GLUU_CONFIG_CONSUL_HOST=consul
      - GLUU_SECRET_VAULT_HOST=vault
      - GLUU_PERSISTENCE_TYPE=ldap
      - GLUU_LDAP_URL=ldap:1636
      # when will next key rotation occurs (in hours)
      - GLUU_KEY_ROTATION_INTERVAL=48
      # checks if oxAuth keys need to be rotated (in seconds)
      - GLUU_KEY_ROTATION_CHECK=3600
    container_name: key-rotation
    volumes:
      - /path/to/vault_role_id.txt:/etc/certs/vault_role_id
      - /path/to/vault_secret_id.txt:/etc/certs/vault_secret_id
    restart: unless-stopped
    labels:
      - "SERVICE_IGNORE=yes"
```
