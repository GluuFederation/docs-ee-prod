## Overview

A common question using a custom LDAP schema in Gluu Server DE containers is when to mount the file and where to put it.
This guide explains how to use custom schema in OpenDJ containers in various scenarios.

## Adding Schema Before Deployment

It is important to know that during the first deployment of the OpenDJ container, files cannot be mounted to `/opt/opendj/config` or the installation will fail. Fortunately, during installation, OpenDJ will copy the schema from `/opt/opendj/template/config/schema` to the `/opt/opendj/config/schema` directory.

Below is an example of how to mount custom schema:

```sh
docker run \
    -v /path/to/78-myAttributes.ldif:/opt/opendj/template/config/schema/78-myAttributes.ldif \
    gluufederation/wrends:4.0.1_02
```

As we can see, `78-myAttributes.ldif` is mounted as `/opt/opendj/template/config/schema/78-myAttributes.ldif` inside the container, which eventually will be copied to `/opt/opendj/config/schema/78-myAttributes.ldif` automatically. This custom schema will be loaded by the OpenDJ server upon startup.

## Adding Schema After Deployment

In this scenario, we assume the container has been running and we need to add a new schema named `79-otherAttributes.ldif`.
Mounting this file into `/opt/opendj/template/config/schema` won't work, as it will not be copied to `/opt/opendj/config/schema` directory inside the container. Instead, we are going to mount the file to `/opt/opendj/config/schema` directly.

```sh
docker run \
    -v /path/to/79-otherAttributes.ldif:/opt/opendj/config/schema/79-otherAttributes.ldif \
    gluufederation/wrends:4.0.1_02
```

!!! Note
    Adding new schema may require restarting the container.

## Custom Schema in Multiple OpenDJ Containers

### Using Docker Config (Swarm Mode)

Create a config file to store the contents of the `78-myAttributes.ldif` custom schema.

```sh
docker config create custom-ldap-schema 78-myAttributes.ldif
```

Mount the schema (depending on the deployment scenario) into the container:

```yaml
# before deployment
services:
  opendj:
    image: gluufederation/wrends:4.0.1_02
    configs:
      - source: custom-ldap-schema
        target: /opt/opendj/template/config/schema/78-myAttributes.ldif

configs:
  custom-ldap-schema:
    external: true
```

Or:

```yaml
# after deployment, restart service if needed
services:
  opendj:
    image: gluufederation/wrends:4.0.1_02
    configs:
      - source: custom-ldap-schema
        target: /opt/opendj/config/schema/78-myAttributes.ldif

configs:
  custom-ldap-schema:
    external: true
```

### Using Kubernetes ConfigMaps

Create a config file to store the contents of the `78-myAttributes.ldif` custom schema.

```sh
kubectl create cm opendj-custom-schema --from-file=78-myAttributes.ldif
```

Mount the schema (depending on deployment scenario) into the container:

```yaml
# before deployment
apiVersion: v1
kind: StatefulSet
metadata:
  name: opendj
spec:
  containers:
    image: gluufederation/wrends:4.0.1_02
    volumeMounts:
      - name: opendj-schema-volume
        mountPath: /opt/opendj/template/config/schema/78-myAttributes.ldif
        subPath: 78-myAttributes.ldif
  volumes:
    - name: opendj-schema-volume
      configMap:
        name: opendj-custom-schema
```

Or:

    # after deployment, restart service if needed
    apiVersion: v1
    kind: StatefulSet
    metadata:
      name: opendj
    spec:
      containers:
        image: gluufederation/wrends:4.0.1_02
        volumeMounts:
          - name: opendj-schema-volume
            mountPath: /opt/opendj/config/schema/78-myAttributes.ldif
            subPath: 78-myAttributes.ldif
      volumes:
        - name: opendj-schema-volume
          configMap:
            name: opendj-custom-schema
