## Overview

The role of the KeyRotation pod is to regenerate `oxauth-keys.jks`. After keys have been regenerated, these keys will be saved into the secrets backend. On the other hand, the oxAuth pod runs periodic task to pull new `oxauth-keys.jks` (if any) from secrets backend.

### Deploying the key-rotation pod

#### Generate `key-rotation.yaml`

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
    kubectl kustomize key-rotation/base > key-rotation.yaml
  ```
  
#### Deploy `key-rotation.yaml`

  ```bash
    kubectl apply -f  key-rotation.yaml
  ```
