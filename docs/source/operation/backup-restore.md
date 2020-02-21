# Backing up data and restoring from backup

## Overview

This guide introduces how to backup data and restore from a backup file.

## Couchbase

### Install backup strategy

A typical installation of Gluu using [`pygluu-kubernetes.pyz`](https://github.com/GluuFederation/enterprise-edition/releases)  will automatiically install a backup strategy that will backup Couchbase every 5 mins to a persistent volume. However, the Couchbase backup can be setup manually:

1.  Download [`pygluu-kubernetes.pyz`](https://github.com/GluuFederation/enterprise-edition/releases). This package can be built [manually](https://github.com/GluuFederation/enterprise-edition/blob/4.1/README.md#build-pygluu-kubernetespyz-manually).

1.  Run :

     ```bash
     ./pygluu-kubernetes.pyz install-couchbase-backup
     ```
     
!!! Note
    `./pygluu-kubernetes.pyz install-couchbase-backup` will not install couchbase.

### Uninstall backup strategy

A file named `couchbase-backup.yaml` will have been generated during installation of backup strategy. Use that as follows to remove the backup strategy:

```bash
kubectl delete -f ./couchbase-backup.yaml
```

### Restore from backup

!!! Note
    An existing Gluu setup must exist for this to work. Please do not attempt to delete any resources and be very careful in handling Gluu configurations and secrets.

#### Couchbase restore step

1.  Install a new Couchbase if needed.

    ```bash
    ./pygluu-kubernetes.pyz install-couchbase
    ```

1.  Create a pod definition file called `restore-cb-pod.yaml` and paste the below yaml changing the `volumes`, `volumeMounts` and `namespace` if they are different. 

    !!! Note
        `./pygluu-kubernetes.pyz install-couchbase-backup` uses the `volumes` and `volumeMounts` as seen in the yaml below
        
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: restore-node
      namespace: cbns
    spec:  # specification of the pod's contents
      containers:
        - name: restore-pod
          image: couchbase/server:enterprise-6.5.0
          # Just spin & wait forever
          command: [ "/bin/bash", "-c", "--" ]
          args: [ "while true; do sleep 30; done;" ]
          volumeMounts:
            - name: "couchbase-cluster-backup-volume"
              mountPath: "/backups"
      volumes:
        - name: couchbase-cluster-backup-volume
          persistentVolumeClaim:
            claimName: backup-pvc
      restartPolicy: Never
    ```

1.  Apply `restore-cb-pod.yaml`.

    ```bash
    kubectl apply -f  restore-cb-pod.yaml
    ```
    
1.  Access the `restore-node` pod.

    ```bash
    kubectl exec -it restore-node -n cbns -- /bin/bash
    ```
 
1.  Choose the backup of choice

    ```bash
    cbbackupmgr list --archive /backups --repo couchbase
    ```
    
    We will choose the oldest we received from the command above `2020-02-20T10_05_13.781131773Z`
    
1.  Preform the restore using the `cbbackupmgr` command.

    ```bash
    cbbackupmgr restore --archive /backups --repo couchbase --cluster cbgluu.cbns.svc.cluster.local --username admin --password passsword --start 2020-02-20T10_05_13.781131773Z --end 2020-02-20T10_05_13.781131773Z
    ```
    
    Learn more about  [`cbbackupmgr`](https://docs.couchbase.com/server/current/backup-restore/cbbackupmgr-restore.html) command and its options.
    
1. Once done delete the `restore-node` pod.

    ```bash
    kubectl delete -f restore-cb-pod.yaml -n cbns
    ```
    
#### Gluu restore step

1.  Download [`pygluu-kubernetes.pyz`](https://github.com/GluuFederation/enterprise-edition/releases). This package can be built [manually](https://github.com/GluuFederation/enterprise-edition/blob/4.1/README.md#build-pygluu-kubernetespyz-manually).

1.  Run :

     ```bash
     ./pygluu-kubernetes.pyz restore
     ```

## OpenDJ / Wren:DS

### Install backup strategy

A typical installation of Gluu using [`pygluu-kubernetes.pyz`](https://github.com/GluuFederation/enterprise-edition/releases)  will automatiically install a backup strategy that will backup opendj / wren:ds every 10 mins `/opt/opendj/ldif`. However, the couchbase backup can be setup manually:

1.  Download [`pygluu-kubernetes.pyz`](https://github.com/GluuFederation/enterprise-edition/releases). This package can be built [manually](https://github.com/GluuFederation/enterprise-edition/blob/4.1/README.md#build-pygluu-kubernetespyz-manually).

1.  Run :

     ```bash
     ./pygluu-kubernetes.pyz install-ldap-backup
     ```
     
!!! Note
    Up to 6 backups will be stored at `/opt/opendj/ldif` on the running `opendj` pod. The backups will carry the name `backup-0.ldif` to `backup-5.ldif` and will be overwritten to save data.

### Uninstall backup strategy

A file named `ldap-backup.yaml` will have been generated during installation of backup strategy. Use that as follows to remove the backup strategy:

```bash
kubectl delete -f ./couchbase-backup.yaml
```

### Restore from backup

!!! Note
    An existing Gluu setup must exist for this to work. Please do not attempt to delete any resources and be very careful in handling Gluu configurations and secrets.

#### OpenDJ / Wren:DS restore step

1.  Opendj volume attached should carry the backups at `/opt/opendj/ldif`

1. If this is a fresh installation , attach the older volume to the new pod.

1.  Access the opendj pod.

    ```bash
    kubectl exec -ti opendj-0 -n gluu /bin/sh
    ```
    
1.  Choose the backup of choice
    ```bash
    ls /opt/opendj/ldif
    ```
1.  Preform the restore using the `import-ldif` command.

    ```bash
    /opt/opendj/bin/import-ldif -n userRoot -l /opt/opendj/ldif/backup-1.ldif
    ```
1.  Run :

     ```bash
     ./pygluu-kubernetes.pyz restore
     ```