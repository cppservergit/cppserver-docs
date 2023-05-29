# API configuration in config.json

The file /etc/cppserver/config.json stores the definition of all services that CPPServer can serve, by default this file is part of the container's image, but you can supply an external file using Kubernetes volumes and configMaps, for more information on those options please check the [QuickStart tutorial](https://github.com/cppservergit/cppserver-docs/blob/main/quickstart.md) in this repository.

![api-definition](https://github.com/cppservergit/cppserver-docs/assets/126841556/cc36e7be-21de-4018-876f-8fe347b62aa9)

As shown above, the config.json file is parsed once by CPPServer on startup, each json record will be transformed to a representation in memory than CPPServer can use in a very fast way to lookup for the target service when a request arrives.

This is is a minimal config.json file, with security and diagnostic services only, no business services.

```
{
	"services":	
	[
		{
			"uri": "/ms/status",
			"function": "getServerInfo",
			"secure": 0
		},
		{
			"uri": "/ms/metrics",
			"function": "getMetrics",
			"secure": 0
		},
		{
			"uri": "/ms/login",
			"function": "login",
			"fields": [
				{"name": "login", "type": "string", "required": "true"},
				{"name": "password", "type": "string", "required": "true"}
			],
			"secure": 0
		},
		{
			"uri": "/ms/logout",
			"function": "logout"
		},
		{
			"uri": "/ms/sessions",
			"function": "getSessionCount",
			"secure": 0
		},					
		{
			"uri": "/ms/version",
			"function": "get_version",
			"secure": 0
		},
		{
			"uri": "/ms/ping",
			"function": "ping",
			"secure": 0
		}
	]
}
```

The file represents an array of JSON objects, each object represents a microservice, that is, a resource accessed via HTTP with GET/POST verb and returns a JSON response. This particular example shown above contains only the base services for health-check, metrics/diagnostics, security and information, all these do not require security, that's why they have the `"secure": 0` attribute. By default all services are secure unless you set this attribute, if the attribute is not present, the service is secure.

The config.json deployed in the QuickStart tutorial contains many more services that will server as examples for your own services, including services for uploding/downloading BLOBs.

## The parts of a service definition

### A service with no parameters

The is the simples form of service:
```
		{
			"db": "db1",
			"uri": "/ms/categ/view",
			"sql": "select * from sp_categ_view()",
			"function": "dbget"
		}
```

As told before, it is secure by default, if you want to test it without login you have to add `"secure": 0` to the attributes.

* db: points to the database defined via environment variable CPP_DB1, when using Kubernetes this is defines as a secret and injected into the variable in the deployment file cppserver.yaml
* uri: the path of the service, this is the lookup key to find the service record in memory
* sql: the SQL command to send to the database, in this case, will retrieve a resultset by invoking a PostgreSQL function, no parameters to inject in the query in this case
* function: the alias of the reusable C++ I/O function that we want to use, in this case `dbget` which returns a JSON array from a resultset returned by the query

That is, this is a microservice definition, that runs as optimized machine language, with no programming effort, just a simple configuration.

This is the response to such a service:
```
{
	"status": "OK",
	"data": [{
		"categ_id": 49,
		"descrip": "Books"
	}, {
		"categ_id": 50,
		"descrip": "C++ and cloud training"
	}, {
		"categ_id": 4,
		"descrip": "Computer and office supplies"
	}, {
		"categ_id": 58,
		"descrip": "Groceries"
	}, {
		"categ_id": 5,
		"descrip": "Health"
	}, {
		"categ_id": 1,
		"descrip": "Pets"
	}, {
		"categ_id": 2,
		"descrip": "Restaurants"
	}]
}
```

### A service with parameters and audit trace

This service will retrieve two resultsets, customer information and the customer's orders, so the response will contain two arrays, the definition of the service:
```
		{
			"db": "db1",
			"uri": "/ms/customer/info",
			"sql": "select * from sp_customer_get($customerid); select * from sp_customer_orders($customerid)",
			"function": "dbgetm",
			"tags": [ {"tag": "customer"}, {"tag": "orders"} ],
			"fields": [
				{"name": "customerid", "type": "string", "required": "true"}
			],
			"audit": {"enabled": "true", "record": "Customer report: $customerid"}
		}
```

* sql: we are using 2 SQL commands, one for the customer information and the other for the customer's orders, both will use the same search parameter `$customerid` which must be defined in the `"fields"` array
* function: this time we use `dbgetm` which can execute multiple SQLs and retrieve a JSON array from each resultset, in contrats with `dbget` which works for a single resultset
* tags: we need to define a name for each array representing the resultset, one for the cursomer and one for the orders
* fields: will contain a definition for each input parameter, there is only one for this case, the customerid, which is string and is required, this imposes a mandatory test, the microservice won't be executed if the supplied parameter does not pass the test, please note that a corresponding marker or placeholder for the SQL attribute will use the same parameter name preceded by the "$" sign, like `$customerid`
* audit: the presence of this attribute is optional, if enabled it will print to the logger (current default implementation) the message specified in the `"reord"`attribute, which can contain the parameters just like in the `"sql"` attribute. This audit trace will contain the origin IP address, the user, the path of the request in addition to the record, the timestamp of the event is managed by the log subsystem (Kubernetes).

The response for this service:
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

If you compare it with the example shown above, you will see that the data field now contains 2 subfields, customer and orders, because that's what we asked in the service definition ("tags" attribute).

