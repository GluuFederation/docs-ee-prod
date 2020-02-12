## Overview

A common question using a custom LDAP schema in Gluu Server EE pods is when to mount the file and where to put it.
This guide explains how to use custom schema in OpenDJ pods in various scenarios.

## Adding Schema Before Deployment

It is important to know that during the first deployment of the OpenDJ pod, files cannot be mounted to `/opt/opendj/config` or the installation will fail. Fortunately, during installation, OpenDJ will copy the schema from `/opt/opendj/template/config/schema` to the `/opt/opendj/config/schema` directory.

Below is an example of how to mount custom schema using kubernetes configmaps:

1. Create a config file to store the contents of the `78-myAttributes.ldif` custom schema.

```sh
kubectl create cm opendj-custom-schema --from-file=78-myAttributes.ldif
```

1. Mount the schema (depending on deployment scenario) into the container:

    ```yaml
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

As we can see, `78-myAttributes.ldif` is mounted as `/opt/opendj/template/config/schema/78-myAttributes.ldif` inside the container, which eventually will be copied to `/opt/opendj/config/schema/78-myAttributes.ldif` automatically. This custom schema will be loaded by the OpenDJ server upon startup.



## Adding Schema After Deployment

In this scenario, we assume the pod has been running and we need to add a new schema named `79-otherAttributes.ldif`.

```yaml
    apiVersion: v1
    kind: StatefulSet
    metadata:
      name: opendj
    spec:
      containers:
        image: gluufederation/wrends:4.0.1_02
        volumeMounts:
          - name: opendj-schema-volume
            mountPath: /opt/opendj/config/schema/79-otherAttributes.ldif
            subPath: 79-otherAttributes.ldif
      volumes:
        - name: opendj-schema-volume
          configMap:
            name: opendj-custom-schema
```
