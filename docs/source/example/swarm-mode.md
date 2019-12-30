## Overview

This an example of running Gluu Server on multiple VMs using [Docker Swarm Mode](https://docs.docker.com/engine/swarm/).

What follows is an explanation of the process we used to deploy clustered Gluu Server Docker containers.

## Requirements

-   Download Manifests

    ```sh
    wget -q https://github.com/GluuFederation/enterprise-edition/archive/4.0.zip \
        && unzip 4.0.zip
    cd enterprise-edition-4.0/examples/swarm/
    ```

-   For **OpenStack** users, take a look at [these](./#notes-for-openstack-users) notes before moving forward

-   For non-OpenStack users:

    1. Install [Docker](https://docs.docker.com/install/)
    1. Install [Docker Machine](https://docs.docker.com/machine/install-machine/)
    1. Get [DigitalOcean access token](https://www.digitalocean.com/community/tutorials/how-to-use-the-digitalocean-api-v2)

## Shared Volume Between Nodes

oxTrust and oxShibboleth rely on a mounted volume to share oxShibboleth configuration files. Given there are three nodes that need to share the same copy of oxShibboleth files, [csync2](http://oss.linbit.com/csync2/) is used. Note, `csync2` is installed as node's OS package, not a container version. The `csync2` setup is executed when running `nodes.sh` script (see section below).

## Provisioning Cluster Nodes

Nodes are divided into two roles:

- Swarm manager, consists of `manager` node
- Swarm worker, consists of `worker-1` and `worker-2` node

These nodes are created/destroyed using `docker-machine` and are deployed as DigitalOcean droplets.

Refer to https://docs.docker.com/engine/swarm/key-concepts/#nodes for an overview of each role.

### Set Up Nodes

We need to create a file containing DigitalOcean access token:

```sh
echo $DO_TOKEN > digitalocean-access-token
```

To set up nodes, execute the command below:

```sh
./nodes.sh up
```

This command will create `manager`, `worker-1`, and `worker-2` nodes and set up the Swarm cluster.

Wait until all processes are completed, and then we can execute this command to check nodes status:

```sh
docker-machine ssh manager 'docker node ls'
```

This will return an output similar to the example below:

```text
ID                          HOSTNAME    STATUS  AVAILABILITY    MANAGER STATUS  ENGINE VERSION
hylmq7086sr1oxmac6k8mjtcr * manager     Ready   Active          Leader          18.03.0-ce
hdvgmuvwm9z5p4u4740ichd77   worker-1    Ready   Active                          18.03.0-ce
hdvgmuvwm9z5p4u4740ichd88   worker-2    Ready   Active                          18.03.0-ce
```

After the Swarm cluster is created, we also create a custom network called `gluu`. To inspect the network, run the following command:

```sh
docker-machine ssh manager 'docker network inspect gluu'
```

This will return the network information:

```json
[
    {
        "Name": "gluu",
        "Id": "j6gt3o0jrgyk5b10h1hf4gxh8",
        "Created": "2018-04-05T18:30:22.47872588Z",
        "Scope": "swarm",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": []
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": null,
        "Options": {
            "com.docker.network.driver.overlay.vxlanid_list": "4097"
        },
        "Labels": null
    }
]
```

### Tear Down Nodes

To destroy nodes, simply execute the command below (regardless of the nodes driver):

```sh
./nodes.sh down
```

This will prompt the user to destroy each node.

## Deploying Services

Basically, a service can be seen as tasks executed on a manager or worker node.
A service manages specific image tasks (such as create/destroy/scale/etc).

Refer to https://docs.docker.com/engine/swarm/key-concepts/#services-and-tasks for an overview of Swarm service.

In this example, the following services/containers are used to deploy the Gluu stack:

- Consul service
- Vault service
- Registrator service
- config-init container
- WrenDS (a fork of OpenDJ) container
- oxAuth service
- oxTrust service
- oxShibboleth service
- oxPassport service
- NGINX service

### Deploy Consul

To deploy the service:

```sh
# connect to remote docker engine in manager node
eval $(docker-machine env manager)
docker stack deploy -c consul.yml gluu
```

### Deploy Vault

The following files are required for Vault auto-unseal process using GCP KMS service:

- `gcp_kms_creds.json`
- `gcp_kms_stanza.hcl`

Obtain Google Cloud Platform KMS credentials JSON file, save it as `gcp_kms_creds.json`. Save the content as Docker secret.

```sh
docker secret create gcp_kms_creds gcp_kms_creds.json
```

Create `gcp_kms_stanza.hcl`:

```
seal "gcpckms" {
    credentials = "/vault/config/creds.json"
    project     = "<PROJECT_NAME>"
    region      = "<REGION_NAME>"
    key_ring    = "<KEYRING_NAME>"
    crypto_key  = "<KEY_NAME>"
}
```

Make sure to adjust the values above, then save the content as Docker secret.

```sh
docker secret create gcp_kms_stanza gcp_kms_stanza.hcl
```

Create Docker config for custom Vault policy:

```sh
docker config create vault_gluu_policy vault_gluu_policy.hcl
```

To deploy the service:

```sh
docker stack deploy -c vault.yml gluu
```

Vault must be initialized (once) and configured to allow containers accessing the secrets:

```sh
./init-vault.sh
```

### Prepare cluster-wide config and secret

Cluster-wide config are saved into Consul KV storage and secrets are saved into Vault. All Gluu containers pull these to self-configure themselves.

Run the following command to prepare the config and secrets:

```sh
./config.sh
```

!!!NOTE
    this process may take some time, please wait until the process completed.

### Deploy Cache Storage (OPTIONAL)

By default, the cache storage is set to `NATIVE_PERSISTENCE`. To use `REDIS`, deploy the service first:

    docker stack deploy -c redis.yml gluu

### Deploy LDAP

Run the following command to deploy `ldap` services (manager, worker-1, and worker-2):

```sh
docker stack deploy -c ldap.yml gluu
```

Import initial data into LDAP:

```sh
./persistence.sh
```

### Deploy Registrator

[Registrator](https://gliderlabs.com/registrator/) acts a service registry bridge to watch oxAuth/oxTrust/oxShibboleth/oxPassport container events. The event will be watched and data will be saved into Consul.
This is needed because the NGINX container needs to reconfigure its config whenever those containers are added or removed into/from the cluster.

Run the following command to deploy `registrator` service:

```sh
docker stack deploy -c registrator.yml gluu
```

### Deploy oxAuth, oxTrust, oxShibboleth, oxPassport, and NGINX

Run the following commands to deploy oxAuth, oxTrust, oxShibboleth, and NGINX:

```sh
DOMAIN=$DOMAIN docker stack deploy -c web.yml gluu
```

!!!note
    $DOMAIN is the domain value that's entered when running `./config.sh`

### Enabling oxPassport

Enable Passport support by following the official docs [here](https://gluu.org/docs/ce/authn-guide/passport/#setup-passportjs-with-gluu).

### Enabling key-rotation and cr-rotate (OPTIONAL)

To enable key rotation for oxAuth keys (useful when we have RP) and cr-rotate (to monitor cycled IP address of oxTrust container used for cache refresh), run the following command:

```sh
docker stack deploy -c utils.yml gluu
```

## Load Balancer

With three nodes that run a clustered Gluu Server, it's recommended to deploy an external loadbalancer, for example: NGINX or [DigitalOcean loadbalancer](https://www.digitalocean.com/products/load-balancer/).
The process of deploying an external loadbalancer is out of the scope of this document.

## Notes for OpenStack Users

1. Ignore [Set up nodes](./#set-up-nodes) and [Requirements](./#requirements) section.

1. **Docker machine will not be used**

1. **Ignore using `./node.sh` for deploying**

1. Provision three VMs with at least 8GB RAM each with OS flavor of your choosing that being ubuntu 18, CentOS7 ...etc

1.  Make sure the ports associated with installation are open. Including docker swarm ports. In CentOS that might mean adding the allowed VMs and their respective ports.

    ```sh
    iptables -N GLUU
    iptables -A GLUU -s 104.130.198.53 -j ACCEPT  # VM docker-swarm-node-1
    iptables -A GLUU -s 104.130.217.164 -j ACCEPT # VM docker-swarm node-2
    iptables -A GLUU -s 104.130.217.190 -j ACCEPT # VM docker-swarm-node-3
    # SWARM PORTS
    iptables -I INPUT -p tcp --dport 2376 -j GLUU
    iptables -I INPUT -p tcp --dport 2377 -j GLUU
    iptables -I INPUT -p tcp --dport 7946 -j GLUU
    iptables -I INPUT -p udp --dport 7946 -j GLUU
    iptables -I INPUT -p udp --dport 4789 -j GLUU
    # SSL PORT
    iptables -I INPUT -p tcp --dport 443 -j GLUU
    #or
    iptables -A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
    ####
    iptables -A GLUU -j LOG --log-prefix "unauth-swarm-access"
    ```

1. Please note that networking is **key** for a successful installation. Before initializing the setup make sure your nodes can reach each other on the ports used for gluu i.e **attach the security policies that allow inner communications between the nodes**.

1. Install latest [Docker](https://docs.docker.com/install/) on each VM.

1.  Initialize Docker swarm mode for your first VM here that would be `Node-1`

    ```sh
    # example: docker swarm init --advertise-addr 104.130.198.53
    docker swarm init --advertise-addr <ip-for-node-1>
    ```

1.  The above command will initialize Docker swarm and output a command that will be used to add workers to this manager node.

    ```text
    To add a worker to this swarm, run the following command:

    docker swarm join \
        --token SWMTKN-1-3pu6hszjassdfsdfknkj234u2938u4jksdfnl-1awxwuwd3z9j1z3puu7rcgdbx \
        104.130.198.53:2377
    ```

1. Execute the output command on the other nodes here `Node-2` and `Node-3`.

    ```sh
    docker swarm join \
        --token SWMTKN-1-3pu6hszjassdfsdfknkj234u2938u4jksdfnl-1awxwuwd3z9j1z3puu7rcgdbx \
        104.130.198.53:2377
    ```

1. **Optional** If you want all of the nodes to be managers promote them using the following command:

    ```sh
    docker node promote node1 node2
    ```

1. Create the network.

    ```sh
    docker network create -d overlay gluu
    ```

1. Create volumes for `Node-1`

    ```bash
    mkdir -p /opt/config-init/db \
        /opt/opendj/config /opt/opendj/db /opt/opendj/ldif /opt/opendj/logs /opt/opendj/backup /opt/opendj/flag \
        /opt/consul \
        /opt/vault/config /opt/vault/data /opt/vault/logs \
        /opt/shared-shibboleth-idp
    ```

1. Create volumes for `Node-2` and `Node-3`

    ```sh
    mkdir -p /opt/opendj/config /opt/opendj/db /opt/opendj/ldif /opt/opendj/logs /opt/opendj/backup \
        /opt/consul \
        /opt/vault/config /opt/vault/data /opt/vault/logs \
        /opt/shared-shibboleth-idp
    ```

1. Continue [deploying services](#deploying-services). But note that you will not use any `docker-machine` commands, but instead execute the commands directly on the respective node here `Node-1` is the manager.
