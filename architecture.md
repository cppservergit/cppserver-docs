# Architecture and deployment with Kubernetes - DRAFT

CPPServer is a compact, optimized native-code application for Linux, it can run as a SystemD service, even in cluster mode on several VMs or baremetal, always behind a Proxy/Load Balancer that provides the TLS endpoint and L7 protections, but that's a traditional approach, outdated.
Now it's all about containers and orchestators, docker for containers' images and Kubernetes as the orchestator that provides all the cluster services: networking, automatic scalability, self-healing, observability and much more, and of course CPPServer was born ready for Kubernetes, from laptop to mainstream Cloud services like Azure or AWS.

## General process architecture

![general-model](https://github.com/cppservergit/cppserver-docs/assets/126841556/cb1355ba-0f7a-4113-9a55-875d8de9f1a7)

CPPServer accepts plain HTTP 1.1 GET/POST requests and returns JSON responses (and also file attachments for BLOB downloads), and Ingress (Proxy/Load Balancer) will provide secure access to CPPServer, which in turn will execute SQL queries and return resultsets as JSON arrays, or plain status codes when no resultset is required. It is a recommended practice to use a layer of SQL procedures/functions in your database to provide CPPServer with callable object, and use a specific credential/database role for CPPServer that only grants access to these objects, that enhances security and may also benefit performance. Do not give CPPServer direct access to tables, only to procedures and functions. CPPServer requires callable object that return resultsets or nothing at all, it's black & white, it does not support output values, resultsets only, so it may be the case on some scenarios were you can provide a wrapper callable object which in turn calls a legacy procedure, and returns the output as CPPServer expects. When using SQLServer it is very important to use `SET NOCOUNT ON` when returning resultsets.
