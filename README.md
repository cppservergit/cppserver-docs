# CPPServer documentation repository

## Overview

CPPServer is a high-performance declarative microservices engine for PostgreSQL/SQLServer [written in C++](https://github.com/cppservergit/cppserver-pgsql), compiled to optimized machine code, for Kubernetes and Cloud serverless services like Azure Container Apps. A single container can serve thousands of microservices in a secure way. Incorporates built-in observability features like JSON stderr logs, direct push to Loki, Prometheus end-point for metrics and tracing.

![api-definition](https://github.com/cppservergit/cppserver-docs/assets/126841556/23fe1d10-549a-47b8-8ca3-53dc8a4376e3)

It allows to create a microservice using JSON, the engine will translate that into a call to a C++ function which in turn calls the native PostgreSQL client API for C, no programming required, easy to test using curl or a browser, plain simple and fast:

```
{
	"db": "db1",
	"uri": "/ms/customer/info",
	"sql": "select * from sp_customer_get($customerid); select * from sp_customer_orders($customerid)",
	"function": "dbgetm",
	"tags": [ {"tag": "customer"}, {"tag": "orders"} ],
	"fields": [
		{"name": "customerid", "type": "string", "required": "true"}
	]
}
```

Request:
```
https://cppserver.com/ms/customer/info?customerid=ANATR
```

JSON Response:
```
{
	"status": "OK",
	"data": {
		"customer": [{
			"customerid": "ANATR",
			"contactname": "Ana Trujillo",
			"companyname": "Ana Trujillo Emparedados y helados",
			"city": "México D.F.",
			"country": "Mexico",
			"phone": "(5) 555-4729"
		}],
		"orders": [{
			"orderid": 10308,
			"orderdate": "1994-10-19",
			"shipcountry": "Mexico",
			"shipper": "Federal Shipping",
			"total": 88.8
		}, {
			"orderid": 10625,
			"orderdate": "1995-09-08",
			"shipcountry": "Mexico",
			"shipper": "Speedy Express",
			"total": 479.75
		}, {
			"orderid": 10759,
			"orderdate": "1995-12-29",
			"shipcountry": "Mexico",
			"shipper": "Federal Shipping",
			"total": 320
		}, {
			"orderid": 10926,
			"orderdate": "1996-04-03",
			"shipcountry": "Mexico",
			"shipper": "Federal Shipping",
			"total": 514.4
		}]
	}
}
```

Internally CPPServer is an EPOLL event-based async server, with one thread CPPServer can serve thousands of connections, it does use a configurable worker-threads pool (default size is 4 threads) to run in background the microservices meanwhile the main thread keeps processing requests. It has been tested with up to 20000 concurrent users bombarding the server, in standalone mode (systemd service), as a container in a Kubernetes cluster behind an Ingress, and as a Container App in Azure.

![thread-pool](https://github.com/cppservergit/cppserver-docs/assets/126841556/bd39274c-603e-4d61-9b57-83154980ea45)

CPPServer is vendor-agnostic, it was designed and built to run from Laptops to On-Prem HA clusters and Cloud managed serverless services.

![container image structure](https://github.com/cppservergit/cppserver-docs/assets/126841556/093b88cd-74fe-444f-9f25-081401af3035)

## Resources:

### Kubernetes
* [Quickstart tutorial in 10 minutes with Kubernetes](https://github.com/cppservergit/cppserver-docs/blob/main/quickstart.md)
* [Administrative Tasks with MicroK8s and CPPServer](https://github.com/cppservergit/cppserver-docs/blob/main/admin-tasks.md)
* [Automatic Kubernetes-native security check with Trivy and MicroK8s](https://github.com/cppservergit/cppserver-docs/blob/main/security-check.md)
* [Apply NetworkPolicy with CPPServer and MicroK8s clusters](https://github.com/cppservergit/cppserver-docs/blob/main/networkpolicy.md)

### Foundation
* [Anatomy of a C++ microservice](https://github.com/cppservergit/cppserver-docs/blob/main/microservice-anatomy.pdf)
* [JSON response structure specification](https://github.com/cppservergit/cppserver-docs/blob/main/json_response_spec.pdf)
* [Security model](https://github.com/cppservergit/cppserver-docs/blob/main/security-model.md)
* [Technical Overview](https://github.com/cppservergit/cppserver-docs/blob/main/tech-overview.md)
* [How to create a microservice with JSON](https://github.com/cppservergit/cppserver-docs/blob/main/json-api-config.md)

### Repos
* [DockerHub repository](https://hub.docker.com/r/cppserver/pgsql)
* [Source Code repo and build instructions](https://github.com/cppservergit/cppserver-pgsql)
