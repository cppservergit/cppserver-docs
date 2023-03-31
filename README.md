# CPPServer documentation repository

## Quick Intro

CPPServer is a high-performance declarative microservices engine for PostgreSQL/SQLServer written in C++, compiled to optimized machine code, for Kubernetes and Cloud serverless services like Azure Container Apps. A single container can serve thousands of microservices in a secure way. Incorporates built-in observability features like JSON stderr logs, direct push to Loki, Prometheus end-point for metrics and tracing.

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

Internally CPPServer is an EPOLL event-based async server, with one thread can server thousands of connections, it does use a configurable worker-threads pool (default size is 4 threads) to run in background the microservices meanwhile the main thread keeps processing requests.

## Resources:

* [Overview](https://cppserver.com)
* [QuickStart in 10 minutes with Kubernetes](https://www.linkedin.com/pulse/quickstart-con-cppserver-y-kubernetes-mart%C3%ADn-c%C3%B3rdova/)
* [Anatomy of a C++ microservice](https://cppserver.com/docs/microservice-anatomy.pdf)
* [DockerHub repository](https://hub.docker.com/r/cppserver/pgsql)

