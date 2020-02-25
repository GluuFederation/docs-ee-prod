# Build pygluu-kubernetes installer

## Overview
[`pygluu-kubernetes.pyz`](https://github.com/GluuFederation/enterprise-edition/releases) is periodically released and does not need to be built manually. However, the process of building the installer package is listed [below](#build-pygluu-kubernetespyz-manually).

## Build `pygluu-kubernetes.pyz` manually

## Prerequisites

1.  Python 3.6+.
1.  Python `pip3` package.

## Installation

### Standard Python package

1.  Create virtual environment and activate:

    ```sh
    python3 -m venv .venv
    source .venv/bin/activate
    ```

1.  Install the package:

    ```
    make install
    ```

    This command will install executable called `pygluu-kubernetes` available in virtual environment `PATH`.

### Python zipapp

1.  Install [shiv](https://shiv.readthedocs.io/) using `pip3`:

    ```sh
    pip3 install shiv
    ```

1.  Install the package:

    ```sh
    make zipapp
    ```

    This command will generate executable called `pygluu-kubernetes.pyz` under the same directory.

### Known bug

- Bug in line 101   File `/Library/Frameworks/Python.framework/Versions/3.8/lib/python3.8/site-packages/kubernetes/client/models/v1beta1_custom_resource_definition_status.py`, line 101, in conditions. The error will look similar to the following:
  
  ```bash
    File "/root/.shiv/pygluu-kubernetes_3e5bddf4d309be28790a1b035ab5d72d0b9f33dfaade59da1bb9ec0bcd0165a4/site-packages/kubernetes/client/models/v1beta1_custom_resource_definition_status.py", line 54, in __init__
    self.conditions = conditions
  File "/root/.shiv/pygluu-kubernetes_3e5bddf4d309be28790a1b035ab5d72d0b9f33dfaade59da1bb9ec0bcd0165a4/site-packages/kubernetes/client/models/v1beta1_custom_resource_definition_status.py", line 101, in conditions
    ValueError: Invalid value for `conditions`, must not be `None`
  ```
  To fix this error just rerun the installation command `./pygluu-kubernetes.pyz <command>` again.
