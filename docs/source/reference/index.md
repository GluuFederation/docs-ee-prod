## Operating System

Size was a major consideration in the base Docker image we used for our containers. Because of that, we use Alpine Linux in most of our Docker images, which comes in at 5 MB by default. Based on comparisons between other base images, we've saved roughly 50-70% of space utilizing Alpine. The average size of each of our containers is about 269 MB due to dependencies.

## Startup

Docker containers generally have entrypoint scripts to prepare templates, configure files, run services, or anything else you need to run to properly initialize a container and run the process. For our containers, we pull most of our files and certificates from config and secret backends.

Because there is a heirarchy of function to Gluu Server, the startup order is managed by custom scripts (originally designed by one of the contributors, Torstein Krause Johansen/@skybert), to ensure the containers don't begin their launch processes until the services superior to the container are fully started. However, there is a time limit, so a container dependent upon another container could fail if those "health checks" aren't being met.

## Networking Considerations

To ensure the Gluu Server can be accessed publicly, ports 80 and 443 must be published to the host server. Both are web server ports.

oxTrust is an OpenID Connect client, so its container is dependent upon oxAuth's `/.well-known/openid-configuration` endpoint, which is only accessible if web server is started. So if the oxTrust container cannot navigate to `https://<hostname>/.well-known/openid-configuration`, it will fail to finish initialization. The container will most likely not exit.

## Tini as Init for Container

Almost all Gluu Server DE images use [Tini](https://github.com/krallin/tini) to handle signal forwarding and to reap processes.
We decided to include `tini` in the build to ensure all container schedulers/orchestrators have an easy way to configure signal forwarding and process reaping.
