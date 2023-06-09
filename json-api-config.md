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

The config.json deployed in the QuickStart tutorial contains many more services that will serve as examples for your own services, including services for uploding/downloading BLOBs.

## The parts of a service definition

### Simple service with no parameters

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

### Service with parameters and audit trace

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

### Service to save data, with security restrictions

This service will invoke a procedure to insert a row in a table, using multiple input parameters, you can see the use of different data types, all fields are required, there is also an authorizarion restriction, only users with any of the roles (can_insert, admin) will be able to execute this service, it's not enough to have a valid security session, the user must have any of those roles associated to the security session, this `user->roles` relation is configured in the underlying security system, this is external to CPPServer, but this information must be obtained during login using any of the CPPServer login adapters. The role names must exist in the underlying security system.
```
		{
			"db": "db1",
			"uri": "/ms/gasto/add",
			"sql": "call sp_gasto_insert($fecha, $categ_id, $monto, $motivo)",
			"function": "dbexec",
			"fields": [
				{"name": "monto", "type": "double", "required": "true"},
				{"name": "fecha", "type": "date", "required": "true"},
				{"name": "motivo", "type": "string", 	"required": "true"},
				{"name": "categ_id", "type": "int", "required": "true"}
			],
			"roles": [{"name":"can_insert"}, {"name":"admin"}]
		}
```
In this case the function used is `dbexec` which executes a data modification query, and won't retrieve a resultset.
There is a special field marker that you can use in your SQL template: $userlogin, it will be replaced by the current user login name, this is retrieved from the security session, this way you can insert or update this value in a given column, like for audit or ownership purposes.

### Service with validation rule

Is this example, we are using plain SQL, not a good practice, we should use procedures or functions that encapsulate the access to the database object, it does provide better security, but putting that aside, the interesting part is the `validator`.
It will use the given function (db_nomatch in this case) and replace input parameters into the given SQL. For this example, this service will delete a row by it PK, but before we do that, we want to check if the row is referenced in another table, we know that the referencial integrity rules will protect the database, but we would like to show a specific validation message if this is the case, and not an error message, so the validator runs right after the inputs have been validated, and it must pass the condition, which for this example is "no rows should be returned from the given query", if the validator fails, the microservice is not executed and a JSON response with status INVALID in returned, you can configure whatever you want for id and description in the validtor attributes, in this case those were defined for the Web frontend included with CPPServer's tutorial.

```
		{
			"db": "db1",
			"uri": "/ms/categ/delete",
			"sql": "delete from demo.categ where categ_id = $categ_id",
			"function": "dbexec",
			"fields": [
				{"name": "categ_id", "type": "integer", "required": "true"}
			],
			"validator": { "function": "db_nomatch", "sql": "SELECT categ_id FROM demo.gasto where categ_id = $categ_id limit 1", "id": "_dialog_", "description": "$err.delete" },
			"roles": [{"name":"sysadmin"}, {"name":"can_delete"}]
		}
```

Usually a microservice that does not return one or more resultsets, will return a JSON response with status OK, like this:
```
{"status": "OK"}
```

Regarding the validator, there is another function called `db_match` which validates the opposite, if the SQL does not return rows then fail, with these two validators `db_nomatch` and `db_match` and the proper SQL commands many common cases can be tested. 

As told before, all SQL attributes, for services and validators, should rely on invoking SQL functions and procedures, and the database role (user credential) used by CPPServer should only have access to execute a set of SQL functions and procedures, for security reasons and performance too, especially in the case of DBMS systems like SQL Server that pre-compile these objects.

![datbase-role](https://github.com/cppservergit/cppserver-docs/assets/126841556/77b10fb3-c869-4758-9eae-229d5965fb2e)

### Sending email after execution of service

CPPServer uses the library libcurl to send email using a secure SMTP server, like Google SMTP Server, you can use a gmail or google workspace account to send secure, content-rich email using Google's Server, but also any other TLS/SMTP server available to your Pods' network.

![email-config](https://github.com/cppservergit/cppserver-docs/assets/126841556/fc7765e0-1fec-4c7d-8d2f-148424781c3d)

This is the fragment of configuration that must be added to the definition of a service:

```
			"email": {	"enabled": "true", 
						"template": "/var/mail/upload-msg.html", 
						"to": "cppserver@martincordova.com", 
						"cc": "", 
						"subject": "Document upload notification", 
						"attachment": "/var/blobs/$document", 
						"attachment-filename": "$filename"
					 }
```

This facility allow to send email after a successful execution of a service, the body will be HTML, you can insert field (input parameters) markers like $customerid into the html template body. The location of these templates should be a volume path available to all your Pods on any node, in a standalone cluster it can be a hostPath volume. 

The configuration was designed to be able to map most common simple scenarios, in its present implementation it supports a single recipient for TO and CC fields, no batch dispatching yet, although one of the recipients could be an address configured to send the email to many recipients. The example above will send an email to a fixed recipient, attaching the just uploaded document, along with some information in the body (the user that uploaded the document, the document's filename, etc).

#### Environment variables

The following environment variables must be passed to the Pod, values shown here are for example only:

* CPP_MAIL_SERVER="smtp://smtp.gmail.com:587"
* CPP_MAIL_USER="notifications@martincordova.com"
* CPP_MAIL_PWD="kjghssiirkcaozyuzt"

The password `CPP_MAIL_PWD` should be injected into the environment variable as a Kubernetes secret, you could do the same for the user's email `CPP_MAIL_USER`, this will be the address used in the `FROM` header of the message.

The message will be sent as multipart mime with HTML body regardless if it does include an attachment or not. In the current implementation only one attachment can be defined, CPPServer's modern C++ API for libcurl supports multiple attachments, but this has not been enabled in the JSON configuration yet.

#### Configuration options

* enabled: true/false, if false then the configuration will be ignored and no email will be sent
* template: the path to the html template, this file can contain field markers like ´$contactname´ of any field defined for the service
* to: recipient, it can be a static address like `cppserver@martincordova.com`, the user's email `$usermail` obtained from the security session, and any of the input fields defined for the service, like `$patient_email`
* cc: same rules as above
* subject: the mail's subject text, no variables allowed in this field
* attachment: optional, the path to the file to be sent as an attachment, in case the file is a blob, like in the example avove, its filename won't be descriptive of its content's type (it's an UUID), in that case you can use the next field
* attachment-filename: optional, a filename with extension that can be used to set the content-type of the attachment, it can be a field marker like `$filename` if that field contains the real filename, like in the example above. This should be used in case the `attachment` attribute is not enough to describe the content type of the attachment, like in the case of a blob whose filename is a UUID like `934-ue88-ksj8fkkm9-xx09`

#### Complete example with email configuration

```
		{
			"db": "db1",
			"uri": "/ms/blob/add",
			"sql": "insert into demo.blob (document, filename, content_type, content_len, title) values ($document, $filename, $content_type, $content_len, $title)",
			"function": "dbexec",
			"fields": [
				{"name": "title", "type": "string", "required": "true"},
				{"name": "document", "type": "string", "required": "true"},
				{"name": "filename", "type": "string", "required": "true"},
				{"name": "content_type", "type": "string", "required": "true"},
				{"name": "content_len", "type": "integer", "required": "true"}
			],
			"audit": {"enabled": "false", "record": "Upload: $filename, $content_len"},
			"email": {	"enabled": "true", 
						"template": "/var/mail/upload-msg.html", 
						"to": "cppserver@martincordova.com", 
						"cc": "$usermail", 
						"subject": "Document upload notification", 
						"attachment": "/var/blobs/$document", 
						"attachment-filename": "$filename"
					 }
		}
```

#### Message body HTML template example

You can see in this template how field markers can be injected into the mail's body, the complexity of the HTML code used is up to you, images should be included from their original internet locations.

```
<html>
  <body>
        <h1>Document Upload Notice</h1>
        <h2>Demo App</h2>
        <p>The following document was uploaded by user: <b>$userlogin</b></p>
        <p>
                <b>File:</b> $filename<br/>
                <b>Size:</b> $content_len bytes<br/>
                <b>Title:</b> $title<br/>
        </p>
  </body>
</html>
```

### Examples from Demo App

The QuickStart tutorial installs a static webapp that is the frontend for the microservices included in config.json, it does include support for upload/download of blobs (documents), please take a look at that file, it's a good source of many examples that cover several common business cases, the same goes for the frontend webapp, this is a web-responsive framework in itself that has the added value that can be easily converted into a native mobile App using Apache Cordova.

Shown here a subset of services, the ones that deal with blobs (there is a corresponding frontend module that invokes them):

```
		{
			"db": "db1",
			"uri": "/ms/blob/add",
			"sql": "insert into demo.blob (document, filename, content_type, content_len, title) values ($document, $filename, $content_type, $content_len, $title)",
			"function": "dbexec",
			"fields": [
				{"name": "title", "type": "string", "required": "true"},
				{"name": "document", "type": "string", "required": "true"},
				{"name": "filename", "type": "string", "required": "true"},
				{"name": "content_type", "type": "string", "required": "true"},
				{"name": "content_len", "type": "integer", "required": "true"}
			],
			"audit": {"enabled": "true", "record": "Upload: $filename, $content_len"}
		},
		{
			"db": "db1",
			"uri": "/ms/blob/view",
			"sql": "select blob_id, title, content_type from demo.blob order by blob_id",
			"function": "dbget"
		},
		{
			"db": "db1",
			"uri": "/ms/blob/delete",
			"sql": "delete from demo.blob where blob_id = $blob_id",
			"function": "deleteFile",
			"fields": [
				{"name": "blob_id", "type": "integer", "required": "true"}
			]
		},
		{
			"db": "db1",
			"uri": "/ms/blob/download",
			"sql": "select document,content_type,filename from demo.blob where blob_id = $blob_id",
			"function": "download",
			"fields": [
				{"name": "blob_id", "type": "integer", "required": "true"}
			]
		},
```

## Review of all configuration options

This example has all the configurable features for a service API, fields, validator, authorized roles and audit trace:

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
			"audit": {"enabled": "true", "record": "Customer report: $customerid"},
			"roles": [{"name":"access_customers"}, {"name":"admin"}],
			"validator": { "function": "db_nomatch", "sql": "select * from sp_black_list($customerid)", "id": "_dialog_", "description": "$err.notavail" },
		}
```

Please note that the audit trace (if enabled) will be registered only if the microservice was successfully executed.

