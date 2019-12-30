## Overview

Cache refresh requires an IP address, but in container deployments, the IP can be recycled by the scheduler. There needed to be a mechanism to automatically configure which oxTrust container runs the cache refresh. The `cr-rotate` container monitors, activates, and deactivates cache refresh running on a specific oxTrust container.

### Deploying the container

Below is an example of `docker-compose.yml` to deploy the cr-rotate container:

```yaml
services:
  cr-rotate:
    image: gluufederation/cr-rotate:4.0.1_02
    environment:
      - GLUU_CONFIG_CONSUL_HOST=consul
      - GLUU_SECRET_VAULT_HOST=vault
      - GLUU_LDAP_URL=ldap:1636
      - GLUU_CR_ROTATION_CHECK=300
      # get container metadata using Docker API
      # if on Kubernetes, replace 'docker' with 'kubernetes'
      - GLUU_CONTAINER_METADATA=docker
    container_name: key-rotation
    volumes:
      - /path/to/vault_role_id.txt:/etc/certs/vault_role_id
      - /path/to/vault_secret_id.txt:/etc/certs/vault_secret_id
      # if running on Docker scheduler, mount the socket
      # omit this volume when running on Kubernetes
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
    labels:
      - "SERVICE_IGNORE=yes"
```
