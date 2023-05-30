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

![single-node model api](https://github.com/cppservergit/cppserver-docs/assets/126841556/907689fc-c972-4d77-b92b-f6050bfcb7ed)

