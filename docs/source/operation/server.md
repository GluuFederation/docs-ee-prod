## Resource

For production, we recommend using server with a minimum of 8GB of memory and 80GB of disk space.

## Storage Driver

Each Docker installation may have different storage drivers depending on the host's OS. Check the storage driver used by the Docker daemon using `docker info`. Here's an example of the output:

```text
Server Version: 18.06.1-ce
Storage Driver: overlay2
 Backing Filesystem: extfs
 Supports d_type: true
 Native Overlay Diff: true
```

From the output above, the storage driver is set to `overlay2`. Refer to this [doc](https://docs.docker.com/storage/storagedriver/select-storage-driver/) to see supported storage drivers.

By default, each container is set to have 10GB of disk size. For OpenDJ containers, this may not be large enough. There are two ways to increase the container disk size:

1.  Set the option globally in `/etc/docker/daemon.json` (this will be applied to all containers).

    ```json
    {
        "storage-driver": "devicemapper",
        "storage-opts": [
            "dm.basesize=20G"
        ]
    }
    ```

    Make sure the Docker daemon service is restarted.

1.  Set the option locally when running the container.

    ```sh
    docker run --storage-opts dm.basesize=20G gluufederation/wrends:4.0.1_02
    ```

Please note that the ability to change the disk size for container depends on [storage driver](https://docs.docker.com/storage/storagedriver/select-storage-driver/).

## Log Rotation for Docker Container

By default, the log driver is set to `json-file`, which means that a container's log file is written to the disk under the `/var/lib/docker/containers` directory. As a result, when using the default log driver `json-file`, the log file will grow and eventually fill the disk. To mitigate this issue, the administrator can change the [log driver](https://docs.docker.com/config/containers/logging/configure/) or customize the `log-opts` to rotate the container log by creating `/etc/docker/daemon.json`, if it does not already exist. See the example below:

```json
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "10m",
        "max-file": "3"
    }
}
```

The administrator can change how many log files are created per container (`max-file` option) and how big each file is (`max-size` option).

!!! Note
    Any modification in `/etc/docker/daemon.json` requires a service restart, such as `systemctl restart docker`.
