## Overview

Cache refresh requires an IP address, but in container deployments, the IP can be recycled by the scheduler. There needed to be a mechanism to automatically configure which oxTrust pod runs the cache refresh. The `cr-rotate` container monitors, activates, and deactivates cache refresh running on a specific oxTrust pod.

### Deploying the cr-rotate pod

#### Generate `cr-rotate.yaml`

  ```bash
    wget -q https://github.com/GluuFederation/enterprise-edition/archive/4.0.zip
    unzip 4.0.zip
    cd enterprise-edition-4.0/kubernetes/
  ```

- Option 1 `create.sh`:

  ```bash
    bash create.sh
  ```
  
- Option 2 `kustomize`:

  ```bash
    kubectl kustomize cr-rotate/base > cr-rotate.yaml
  ```
  
#### Deploy `cr-rotate.yaml`

  ```bash
    kubectl apply -f  cr-rotate.yaml
  ```
