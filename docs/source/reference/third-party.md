The Gluu Server Docker containers consist of in-house and 3rd-party containers. The 3rd-party containers are the following:

## Consul

!!! Note
    For Kubernetes deployment, this container can be omitted.

The Gluu Server Docker containers were built to be centralized around a config backend. For our use case, we've used [Consul](https://www.consul.io/). The reasoning behind this decision was to allow the containers to be as modular as possible. Services can be replaced without concern for losing any of the default configuration. This isn't to say that there won't need to be persistence using volumes for custom files and long standing data requirements.

## Vault

!!! Note
    For Kubernetes deployment, this container can be omitted.

Similar to Consul, The Gluu Server Docker containers were built to be centralized around a secret backend. For our use case, we've used [Vault](http://vaultproject.io/).

To use the Vault container within Gluu Server Docker Edition, the user needs to initialize and unseal Vault first. The initialization also involves enabling specific role and auth methods. See [Vault operational guide](/operation/vault) for details.

## Registrator

!!! Note
    For Kubernetes deployment, this container can be omitted.

Due to the design of Docker networking where the container IP gets recycled dynamically, the [Registrator](http://gliderlabs.github.io/registrator/latest/) container is used for registering and deregistering oxAuth, oxTrust, oxShibboleth, and oxPassport container IPs. With the help of Registrator, the Nginx container will route the traffic to available oxAuth/oxTrust/oxShibboleth/oxPassport backends.

To connect to Consul container that deployed using HTTPS scheme, please use the `gliderlabs/registrator:master` image instead.
