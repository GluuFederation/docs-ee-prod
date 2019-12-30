## Overview

In a Docker environment where each container can have one or more replicas, it is easier to check the log by collecting all containers' logs, storing them in a single place and possibly searching the logs later. There are tools available to assist in this task, both open source and paid. This guide will show an example of how to collect selected container's logs (oxAuth, oxTrust, OpenDJ, oxShibboleth, oxPassport, and optionally NGINX), using [Filebeat](https://www.elastic.co/products/beats/filebeat), [Elasticsearch](https://www.elastic.co/products/elasticsearch), and [Kibana](https://www.elastic.co/products/kibana).

### Prerequisites

1.  Choose the `json-file` logging driver for the Docker daemon, as Filebeat works best with this driver. By default, the Docker installation uses `json-file` driver, unless set to another driver. Use `docker info | grep 'Logging Driver'` to check current logging driver.
1.  The Elasticsearch container requires the host's specific `vm.max_map_count` kernel setting to be at least 262144. Refer to the official installation page of Elasticsearch [here](https://www.elastic.co/guide/en/elasticsearch/reference/6.6/docker.html#docker-cli-run-prod-mode).

### Logging Containers in Docker/Docker Swarm

1.  Create `filebeat.yml` for custom Filebeat configuration:

    ```yaml
    filebeat.config:
      modules:
        path: ${path.config}/modules.d/*.yml
        reload.enabled: false

    output.elasticsearch:
      hosts: '${ELASTICSEARCH_HOSTS:elasticsearch:9200}'

    filebeat.autodiscover:
      providers:
        - type: docker
          templates:
            - condition:
                regexp:
                  docker.container.image: "ox[auth|trust|shibboleth]"
              config:
                - type: docker
                  containers.ids:
                    - "${data.docker.container.id}"
                  encoding: utf-8
                  enabled: true
                  document_type: docker
                  multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
                  multiline.negate: true
                  multiline.match: after
            - condition:
                contains:
                  docker.container.image: wrends
              config:
                - type: docker
                  containers.ids:
                    - "${data.docker.container.id}"
                  encoding: utf-8
                  enabled: true
                  document_type: docker
                  include_lines: ['^\[']
            - condition:
                contains:
                  docker.container.image: oxpassport
              config:
                - type: docker
                  containers.ids:
                    - "${data.docker.container.id}"
                  encoding: utf-8
                  enabled: true
                  document_type: docker
                  include_lines: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
            - condition:
                contains:
                  docker.container.image: nginx
              config:
                - type: docker
                  containers.ids:
                    - "${data.docker.container.id}"
                  encoding: utf-8
                  enabled: true
                  document_type: docker
                  exclude_files: ['\.gz$']

    processors:
      # decode the log field (sub JSON document) if JSONencoded, then maps it's fields to elasticsearch fields
      - decode_json_fields:
          fields: ["log"]
          target: ""
          # overwrite existing target elasticsearch fields while decoding json fields
          overwrite_keys: true
      - add_docker_metadata: ~
      - add_cloud_metadata: ~

    # Write Filebeat own logs only to file to avoid catching them with itself in docker log files
    logging.to_files: true
    logging.to_syslog: false
    logging.level: warning
    ```

    For a Docker Swarm setup, save the file above into Docker configurations so this config can be injected into multiple Filebeat containers.

1.  Create a Docker manifest file, i.e. `docker-compose.yml`:

    ```yaml
    # The following example is based on `docker-compose` manifest file:
    version: "2.3"

    services:
      filebeat:
        image: docker.elastic.co/beats/filebeat:6.6.1
        command: filebeat -e -strict.perms=false
        container_name: filebeat
        restart: unless-stopped
        user: root
        volumes:
          - /path/to/filebeat/volumes/data:/usr/share/filebeat/data:rw
          - /path/to/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
          - /var/run/docker.sock:/var/run/docker.sock:ro
          - /var/lib/docker/containers:/var/lib/docker/containers:ro
        labels:
          - "SERVICE_IGNORE=yes"

      elasticsearch:
        image: docker.elastic.co/elasticsearch/elasticsearch:6.6.1
        container_name: elasticsearch
        restart: unless-stopped
        environment:
          - cluster.name=gluu-cluster
          - bootstrap.memory_lock=true
          - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
          - "TAKE_FILE_OWNERSHIP=true"
        ulimits:
          memlock:
            soft: -1
            hard: -1
        volumes:
          - /path/to/elasticsearch/volume/data:/usr/share/elasticsearch/data
        labels:
          - "SERVICE_IGNORE=yes"

      kibana:
        image: docker.elastic.co/kibana/kibana:6.6.1
        container_name: kibana
        restart: unless-stopped
        labels:
          - "SERVICE_IGNORE=yes"
    ```

    Adjust the manifest file if Docker Swarm mode is used.

1.  Deploy the containers using `docker-compose up -d` or `docker stack deploy $STACK_NAME` command.

### Logging Containers in Kubernetes

1.  Create `filebeat.yml` for custom Filebeat configuration:

    ```yaml
    filebeat.config:
      modules:
        path: ${path.config}/modules.d/*.yml
        reload.enabled: false

    output.elasticsearch:
      hosts: '${ELASTICSEARCH_HOSTS:elasticsearch:9200}'

    filebeat.autodiscover:
      providers:
        - type: kubernetes
          templates:
            - condition:
                regexp:
                  kubernetes.container.image: "ox[auth|trust|shibboleth]"
              config:
                - type: docker
                  containers.ids:
                    - "${data.kubernetes.container.id}"
                  encoding: utf-8
                  enabled: true
                  document_type: docker
                  multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
                  multiline.negate: true
                  multiline.match: after
            - condition:
                contains:
                  kubernetes.container.image: wrends
              config:
                - type: docker
                  containers.ids:
                    - "${data.kubernetes.container.id}"
                  encoding: utf-8
                  enabled: true
                  document_type: docker
                  include_lines: ['^\[']
            - condition:
                contains:
                  kubernetes.container.image: oxpassport
              config:
                - type: docker
                  containers.ids:
                    - "${data.kubernetes.container.id}"
                  encoding: utf-8
                  enabled: true
                  document_type: docker
                  include_lines: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
            - condition:
                contains:
                  kubernetes.container.image: nginx
              config:
                - type: docker
                  containers.ids:
                    - "${data.kubernetes.container.id}"
                  encoding: utf-8
                  enabled: true
                  document_type: docker
                  exclude_files: ['\.gz$']

    processors:
      # decode the log field (sub JSON document) if JSONencoded, then maps it's fields to elasticsearch fields
      - decode_json_fields:
          fields: ["log"]
          target: ""
          # overwrite existing target elasticsearch fields while decoding json fields
          overwrite_keys: true
      - add_kubernetes_metadata: ~
      - add_cloud_metadata: ~

    # Write Filebeat own logs only to file to avoid catching them with itself in docker log files
    logging.to_files: true
    logging.to_syslog: false
    logging.level: warning
    ```

    Save the content into Kubernetes' ConfigMaps using `kubectl create cm filebeat-config --from-file=filebeat.yml` for later use.

1.  Create a Kubernetes manifest file, `filebeat-roles.yaml`:

    ```yaml
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: filebeat
      namespace: default
    subjects:
    - kind: User
      name: system:serviceaccount:default:default # Name is case sensitive
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: ClusterRole
      name: filebeat-role
      apiGroup: rbac.authorization.k8s.io

    ---

    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: filebeat-role
      namespace: default
    rules:
      - apiGroups: [""] # "" indicates the core API group
        resources:
          - namespaces
          - pods
        verbs:
          - get
          - watch
          - list
    ```

    Afterwards, run `kubectl apply -f filebeat-roles.yaml` to define custom roles and role binding for Filebeat.

1.  Create a Kubernetes manifest file, `elasticsearch.yaml`:

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: elasticsearch
      labels:
        app: elasticsearch
    spec:
      ports:
        - port: 9200
          name: elastic
          targetPort: 9200
          protocol: TCP
        - port: 9300
          name: elastic-cluster
          targetPort: 9300
          protocol: TCP
      selector:
        app: elasticsearch
      clusterIP: None

    ---

    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: elasticsearch
      labels:
        app: elasticsearch
    spec:
      serviceName: elasticsearch
      replicas: 1
      selector:
        matchLabels:
          app: elasticsearch
      template:
        metadata:
          labels:
            app: elasticsearch
        spec:
          containers:
            - name: elasticsearch
              imagePullPolicy: Always
              image: docker.elastic.co/elasticsearch/elasticsearch:6.6.1
              ports:
                - containerPort: 9200
                  name: elastic
                - containerPort: 9300
                  name: elastic-cluster
              env:
                - name: cluster.name
                  value: "gluu-cluster"
                - name: bootstrap.memory_lock
                  value: "false"
                - name: ES_JAVA_OPTS
                  value: "-Xms512m -Xmx512m"
                - name: TAKE_FILE_OWNERSHIP
                  value: "true"
              volumeMounts:
                - name: elasticsearch-data
                  mountPath: /usr/share/elasticsearch/data
        volumes:
          - name: elasticsearch-data
            hostPath:
              path: /data/elasticsearch/data
              type: DirectoryOrCreate

    ```

    Run `kubectl apply -f elasticsearch.yml` to deploy Elasticsearch Pod.

1.  Create a Kubernetes manifest file, `filebeat-ds.yaml`:

    ```yaml
    apiVersion: extensions/v1beta1
    kind: DaemonSet
    metadata:
      name: filebeat
      labels:
        app: filebeat
    spec:
      template:
        metadata:
          labels:
            app: filebeat
        spec:
          terminationGracePeriodSeconds: 30
          containers:
            - name: filebeat
              image: docker.elastic.co/beats/filebeat:6.6.1
              args: [
                "-e",
                "-strict.perms=false",
              ]
              env:
                - name: ELASTICSEARCH_HOST
                  value: elasticsearch
                - name: ELASTICSEARCH_PORT
                  value: "9200"
                - name: ELASTICSEARCH_USERNAME
                  value:
                - name: ELASTICSEARCH_PASSWORD
                  value:
                - name: ELASTIC_CLOUD_ID
                  value:
                - name: ELASTIC_CLOUD_AUTH
                  value:
              securityContext:
                runAsUser: 0
              volumeMounts:
                - name: filebeat-config
                  mountPath: /usr/share/filebeat/filebeat.yml
                  readOnly: true
                  subPath: filebeat.yml
                - name: filebeat-data
                  mountPath: /usr/share/filebeat/data
                - name: varlibdockercontainers
                  mountPath: /var/lib/docker/containers
                  readOnly: true
          volumes:
            - name: filebeat-config
              configMap:
                defaultMode: 0600
                name: filebeat-config
            - name: varlibdockercontainers
              hostPath:
                path: /var/lib/docker/containers
            - name: filebeat-data
              hostPath:
                path: /data/filebeat/data
                type: DirectoryOrCreate
    ```

    Run `kubectl apply -f filebeat-ds.yml` to deploy Filebeat Pod.

1.  Create a Kubernetes manifest file, `kibana.yaml`:

    ```yaml
    apiVersion: v1
    kind: Service
      metadata:
        name: kibana
        labels:
          app: kibana
    spec:
      ports:
        - port: 5601
          name: kibana
      selector:
        app: kibana

    ---

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: kibana
      labels:
        app: kibana
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: kibana
      template:
        metadata:
          labels:
            app: kibana
        spec:
          containers:
            - name: kibana
              image: docker.elastic.co/kibana/kibana:6.6.1
              ports:
                - containerPort: 5601
    ```

    Run `kubectl apply -f kibana.yml` to deploy Kibana Pod.

### Accessing Kibana UI

The examples above don't expose Kibana UI port 5601 for security reasons.
Below are examples of how to access the UI:

1.  Kubernetes

    - expose the port using `kubectl port-forward $KIBANA_POD 5601:5601`
    - use SSH tunneling to get the port locally using `ssh -L 5601:localhost:5601 $USER@$REMOTE_HOST`
    - visit `http://localhost:5601` to access the Kibana UI

1.  Docker/Docker Swarm

    - get the Kibana's container IP
    - visit the browser at `http://$KIBANA_CONTAINER_IP:5601`.

    or for Swarm mode:

    - ensure the Kibana container exposes port 5601 to the host's `loopback` address (`127.0.0.1`), similar to `docker run -p 127.0.0.1:5601:5601 kibana`
    - use SSH tunneling using `ssh -L 5601:localhost:5601 $USER@$REMOTE_HOST`
    - visit `http://localhost:5601` to access the Kibana UI
