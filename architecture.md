# Architecture and deployment with Kubernetes - DRAFT

CPPServer is a compact, optimized native-code application for Linux, it can run as a SystemD service, even in cluster mode on several VMs or baremetal, always behind a Proxy/Load Balancer that provides the TLS endpoint and L7 protections, but that's a traditional approach, outdated.
Now it's all about containers and orchestators, docker for containers' images and Kubernetes as the orchestator that provides all the cluster services: networking, automatic scalability, self-healing, observability and much more, and of course CPPServer was born ready for Kubernetes, from laptop to mainstream Cloud services like Azure or AWS.

## General process architecture

![general-model](https://github.com/cppservergit/cppserver-docs/assets/126841556/cb1355ba-0f7a-4113-9a55-875d8de9f1a7)

CPPServer accepts plain HTTP 1.1 GET/POST requests and returns JSON responses (and also file attachments for BLOB downloads), and Ingress (Proxy/Load Balancer) will provide secure access to CPPServer, which in turn will execute SQL queries and return resultsets as JSON arrays, or plain status codes when no resultset is required. It is a recommended practice to use a layer of SQL procedures/functions in your database to provide CPPServer with callable objects, and use a specific credential/database role for CPPServer that only grants access to these objects, that enhances security and may also benefit performance. Do not give CPPServer direct access to tables, only to procedures and functions. CPPServer requires callable object that return resultsets or nothing at all, it's black & white, it does not support output values, resultsets only, so it may be the case on some scenarios were you can provide a wrapper callable object which in turn calls a legacy procedure, and returns the output as CPPServer expects. When using SQLServer it is very important to use `SET NOCOUNT ON` when returning resultsets.

## CPPServer as a container

CPPServer was designed to be packaged as a container image, it does use environment variables to read its general configuration, an external file config.json (/etc/cppserver/config.json) to read microservices' definitions. Database connection strings are injected into environment variables from Kubernetes secrets, and the config.json file get injected as a Kubernetes configMap and mapped to a volume, all of these are accepted Kubernetes practices, and even on specialized cloud services like Azure Container Apps, there are equivalent configuration options (because these services use Kubernetes as their base).

![container image structure](https://github.com/cppservergit/cppserver-docs/assets/126841556/3a4b8147-46ac-4dea-a773-f9b757e37c88)

It's only dependency in the current version is the PostgreSQL native client API (libpq5), Ubuntu 22.04 LTS was choosen as the image's base for its robustness and security.

The basic dockerfile to build the image:

```
FROM ubuntu:latest
LABEL maintainer="cppserver@martincordova.com"
RUN apt update
RUN apt install -y --no-install-recommends libpq5
RUN apt clean
RUN mkdir /opt/cppserver
RUN mkdir /etc/cppserver
ADD config.json /etc/cppserver
ADD cppserver /opt/cppserver
ENV CPP_HTTP_LOG=0
ENV CPP_POOL_SIZE=4
ENV CPP_PORT=8080
ENV CPP_LOGIN_LOG=0
ENV CPP_STDERR_LOG=1
ENV CPP_LOKI_SERVER=""
ENV CPP_LOKI_PORT=3100
EXPOSE 8080
WORKDIR /opt/cppserver
ENTRYPOINT ["./cppserver"]
```

The dockerfile is veru basic, it does not contain any security-related settings because those are set in the cppserver.yaml file, the deployment descriptor for Kubernetes.
For more information please check the [official CPPServer image repository](https://hub.docker.com/r/cppserver/pgsql) at dockerhub.

## Kubernetes deployment options

There is great flexibility regarding cluster design when deploying with Kubernetes, it depends a lot on the environment, is it local development? testing? production? it is up to you, it can be OnPrem or using a Cloud service, even a Kubernetes cloud service or something like Azure Container Apps that hides most of the complexity of Kubernetes. It's up to you. The focus in this section is mostly on-premise deployments, which can be used for development or production, depending on the workloads expected to be managed by the cluster.

For these on-premise deployments we recommend and support [Canonical's MicroK8s Kubernetes](https://microk8s.io/docs), it's fast, light and pretty stable, also well documented, and runs very well on Ubuntu 22.04 and LXD containers (native Linux containers). All of our documentation related to CPPServer with Kubernetes relies on the use of MicroK8s.

### Single-node Cluster

![arch-1](https://github.com/cppservergit/cppserver-docs/assets/126841556/dc2062be-9f43-4273-b0c6-67d05e428c65)

The most simple setup, suitable for local development but also for testing, depending on the hardware this setup can handle 10000+ concurrent users with a single pod, it can auto-scale to multiple pods, external resources like a static website can be managed in the local filesystem (Kubernetes hostPath storage) and all the Pods will have access to them via a volume mapping, these resources can be updated and refreshed on CPPServer for instant feedback on changes made to these files, which is very handy for frontend development. As for the backend, if config.json is managed as a configMap (instead of using the one stored in the image), then it can be updated without affecting the image, then a deployment rollout will refresh the Pods with the new configMap, in a few seconds, which makes the microservice building/testing cycle very agile.

### High-availability cluster, single VM with LXD

![arch2](https://github.com/cppservergit/cppserver-docs/assets/126841556/cba43be7-f1ea-47de-8236-c36f1ef9a010)

Using LXD to partition a single VM or bare-metal into three native Linux containers (lighter than a full VM) you can setup a high-availability MicroK8s cluster, which requires a minimum of three nodes, you can add more nodes, as worker nodes or stand-by nodes, it dependes on the computing power available on the VM or baremetal machine you are using for the cluster. What's very interesting about this setup is that you can use a single machine to create a high-availability multi-node K8s cluster, similar to a Cloud setup. CPPServer Pods would auto-scale across the nodes, Kubernetes takes care of that and much more. The storage (for the static website) must be shared among the nodes of the cluster, this is transparent to CPPServer as long as the shared storage can be mapped using a Kubernetes volume, you can use an NFS server or a Kubernetes-native cluster wide storage solution.
Check the [MicroK8s clustering guide](https://microk8s.io/docs/clustering) for more information about high-availability.

In order to balance the load between the Kubernetes cluster nodes, HAProxy load balancer will be installed as a native Linux service, on the same host machine, but it will be the only process visible from the external network, the cluster nodes are not visible to the external network, they are on a subnet inside the host machine, which is only visible to HAProxy, there is a natural security feature in this setup as a consequence of using LXD to partition the host machine.

It's relevant to note that HAProxy can use HTTP or HTTPS to contact the Ingress on each node, considering the host is closed to the external network, plain HTTP can be used for better performance.

### High-availability cluster, multiple VMs

![arch-3](https://github.com/cppservergit/cppserver-docs/assets/126841556/e3167d0e-e1bd-4712-befa-80bf89be2641)

This is a variation of the last model, but more expensive in terms of computing power because it does use a separate VM for each cluster node and also for the external HAProxy (load balancer).

### Single-node cluster with replica

![arch-4](https://github.com/cppservergit/cppserver-docs/assets/126841556/1a248c9b-7cfc-4112-9ba0-9c13df07e1b4)

In this model, which is similar to the first one, there is no HA cluster on Kubernetes, nevertheless you can have several Pods running on the node and there is load balancing and self-healing at the Pod level.
What is relevant on this model is that you can have an identical copy of the 1st node, and configure HAProxy as the load balancer with the 2nd node as a backup or stand-by, if the 1st node stops working, HAProxy will start using the 2nd node, it is a way to have high availability on-prem with a couple of simple node setups. External resources like a static website must be placed in shared storage visible to both nodes. Each node is a single-node cluster, there is no relation between them, each one has its own control plane.

## Deployment examples

These are YAML examples for deployment of CPPServer on Kubernetes. They rely on the official CPPServer images stored at dockerhub, nevertheless, you can use a local registry with MicroK8s and suppy your own images, consider that on a zero-trust environment you may require to compile CPPServer by yourself and build your own docker image, and then feed this image to MicroK8s registry. Some of these deployments assume you only need SQL login adapter, so no [LoginServer](https://hub.docker.com/r/cppserver/pgsql-login) will be deployed, otherwise you have to deploy LoginServer with its own YAML. LoginServer provides focused login services to CPPServer and it also includes LDAP as a login option (configurable via environment variables), also if there is need to write very specific C++ code for authentication, having a separate LoginServer helps keep CPPServer isolated from these changes.

![Pod deployment model](https://github.com/cppservergit/cppserver-docs/assets/126841556/842c282a-c00e-4a20-95fb-4678bd9c80ac)

### Single node

Uses hostPath storage (local filesystem), pre-defined secrets and a pre-loaded configMap for config.json. You would have to edit the location for the volumes, one for the static website (if you need it) and another for the blobs storage `/var/blobs`, CPPServer uploads support depends on this path, cannot be changed. This deployment also installs the scheduled task CPPJob, to execute the procedure that removes expired security sessions. This is a simplified deployment that uses CPPServer built-in SQL login adapter, LoginServer is not being deployed in this case. In the Ingress section there is a mapping to a static website /demo in this example, you can remove it if you are not serving static content with CPPServer, the same with the corresponding volume and its mapping. Note that if you are serving static content, the volume mapping must be `/var/www`.
Regarding the creation of secrets and the configMap, it is all explained in the [QuickStart tutorial](https://github.com/cppservergit/cppserver-docs/blob/main/quickstart.md).
```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cppserver
  namespace: cppserver
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cppserver
  template:
    metadata:
      labels:
        app: cppserver
    spec:
      volumes:
      - name: blobs
        hostPath:
          path: /home/ubuntu/blobs
      - name: www
        hostPath:
          path: /home/ubuntu/www
      - name: config
        configMap:
          name: cppserver-config
      containers:
        - name: cppserver
          image: cppserver/pgsql:1.2.0
          livenessProbe:
            httpGet:
              path: /ms/ping
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /ms/ping
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
          volumeMounts:
          - name: blobs
            mountPath: /var/blobs
          - name: www
            mountPath: /var/www
          - name: config
            mountPath: /etc/cppserver
          env:
          - name: CPP_POOL_SIZE
            value: "4"
          - name: CPP_LOGIN_LOG
            value: "0"
          - name: CPP_HTTP_LOG
            value: "0"
          - name: CPP_STDERR_LOG
            value: "1"
          - name: CPP_LOKI_SERVER
            value: ""
          - name: CPP_LOKI_PORT
            value: "3100"
          - name: CPP_DB1
            valueFrom:
              secretKeyRef:
                name: cpp-secret-db1
                key: connstr
                optional: false
          - name: CPP_SESSIONDB
            valueFrom:
              secretKeyRef:
                name: cpp-secret-sessiondb
                key: connstr
                optional: false
          - name: CPP_LOGINDB
            valueFrom:
              secretKeyRef:
                name: cpp-secret-logindb
                key: connstr
                optional: false
          ports:
          - containerPort: 8080
          imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: cppserver
  namespace: cppserver
spec:
  ports:
  - port: 8080
    targetPort: 8080
    name: tcp
  selector:
    app: cppserver
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cppserver
  namespace: cppserver
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /demo
        pathType: Prefix
        backend:
          service:
            name: cppserver
            port:
              number: 8080
      - path: /ms
        pathType: Prefix
        backend:
          service:
            name: cppserver
            port:
              number: 8080
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cppjob
  namespace: cppserver
spec:
  schedule: "*/2 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cppjob
            image: cppserver/pgsql-job:1.00
            imagePullPolicy: Always
            env:
            - name: CPP_STDERR_LOG
              value: "1"
            - name: CPP_LOKI_SERVER
              value: ""
            - name: CPP_LOKI_PORT
              value: "3100"
            - name: CPP_JOB_QUERY
              value: "call cpp_session_timeout()"
            - name: CPP_JOB_DB
              valueFrom:
                secretKeyRef:
                  name: cpp-secret-sessiondb
                  key: connstr
                  optional: false
          restartPolicy: OnFailure
```

#### No Volumes

If you are not going to serve a static website with CPPServer, receive uploads and you prefer to use config.json embedded into the container's image, then you can get rid of the volumes and volumeMounts sections shown above, and you won't need to create a configMap containing config.json prior to applying this YAML.
Regarding config.json, it is a question of convenience, if you are changing config.json frequently (like during development) then you may want to keep it as a configMap and just `rollout restart` CPPServer deployment in order to load the new config.json. In production you may prefer using only an image containing config.json, and whenever there is an update in config.json, you will have to update the image and its tag (optional with dockerhub), update the reference in the YAML and apply it to the cluster, if you are using a local registry you need to change the image's tag (usually a version number) in order to make the deployment use the new image, this is not necessary when referencing the image from dockerhub (default).


### Using NFS volumes

Let's assume there is an NFS server at 192.168.0.222 that exports the following folders:
```
/srv
└── nfs
    ├── blobs
    └── www
        └── demo
            ├── categ.html
            ├── css
            │   ├── frontend.css
            │   ├── highcharts.css
            │   ├── pico.css
            │   └── pico.css.map
            ├── customer.html
            ├── gasto.html
            ├── images
            │   └── apple-icon.png
            ├── index.html
            ├── js
            │   ├── common.js
            │   ├── frontend.js
            │   ├── highcharts-3d.js
            │   ├── highcharts-3d.js.map
            │   ├── highcharts.js
            │   ├── highcharts.js.map
            │   └── locale.js
            ├── menu.html
            ├── products.html
            ├── sales.html
            ├── shippers.html
            └── upload.html
```
And you are using a high-availability cluster with 3+ nodes, and all the nodes need to access the same volumePaths, so when you update the static website, all Pods in all nodes can see the update, and when a user requests to download a blob, the it will be available regardless of the node/pod that served the request. To achieve this you need to be able to map /var/www to 192.168.0.222:/srv/nfs/www  and /var/blobs to 192.168.0.222:/srv/nfs/blobs.

The first step is to install NFS client in every node of the cluster, tipically this happens when you are setting up all the infraestructure:

```
sudo apt update
sudo install nfs-common -y
```

That's all from the operating system side.

The YAML to deploy CPPServer must be changed regarding the definition of the volumes and the volumesPath, although minimal changes, first we define the NFS volume:
```
      volumes:
      - name: nfs-server
        nfs:
          server: 192.168.0.222
          path: /srv/nfs
          readOnly: no
```

Then we simplify the volumeMap, we only need a single one because of the way the NFS server organized its exports:
```
          volumeMounts:
          - name: nfs-server
            mountPath: /var
```

This way /var/blobs and /var/www will point to the right shared folders, and will be available cluster-wide.

