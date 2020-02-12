# Upgrading Versions

## Overview

This guide introduces how to upgrade from one version to another

## Upgrade

1.  Download [`pygluu-kubernetes.pyz`](https://github.com/GluuFederation/enterprise-edition/releases). This package can be built [manually](https://github.com/GluuFederation/enterprise-edition/blob/4.1/README.md#build-pygluu-kubernetespyz-manually).

1.  Run :

     ```bash
     ./pygluu-kubernetes.pyz upgrade
     ```
## Exporting Data

1.  Make sure to backup existing LDAP data

1.  Set environment variable as a placeholder for LDAP server password (for later use):

    ```sh
    export LDAP_PASSWD=YOUR_PASSWORD_HERE
    ```

1.  Assuming that existing LDAP container called `ldap` has data, export data from each backend:

    1.  Export `o=gluu`

        ```sh
        kubectl exec -ti ldap /opt/opendj/bin/ldapsearch \
            -Z \
            -X \
            -D "cn=directory manager" \
            -w $LDAP_PASSWD \
            -p 1636 \
            -b "o=gluu" \
            -s sub \
            'objectClass=*' > gluu.ldif
        ```

    1.  Export `o=site`

        ```sh
        kubectl exec -ti ldap /opt/opendj/bin/ldapsearch \
            -Z \
            -X \
            -D "cn=directory manager" \
            -w $LDAP_PASSWD \
            -p 1636 \
            -b "o=site" \
            -s sub \
            'objectClass=*' > site.ldif
        ```

    1.  Export `o=metric`

        ```sh
        kubectl exec -ti ldap /opt/opendj/bin/ldapsearch \
            -Z \
            -X \
            -D "cn=directory manager" \
            -w $LDAP_PASSWD \
            -p 1636 \
            -b "o=metric" \
            -s sub \
            'objectClass=*' > metric.ldif
        ```

1.  Unset `LDAP_PASSWD` environment variable
