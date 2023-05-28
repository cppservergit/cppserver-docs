# CPPServer Technical Overview (draft - in progress)

CPPServer is a minimal (~300K) http server whose sole purpose is to serve GET/POST requests that will in turn invoke functions to execute SQL queries and return JSON to the clients, which can be any HTTP client, including browsers, mobile apps, another server, etc. It was designed to be a no-code API engine, meaning that you can declare or configure your APIs using a JSON file, and CPPServer will parse this file on startup and serve the requests according to this configuration.

![basic-workflow](https://github.com/cppservergit/cppserver-docs/assets/126841556/f774bbb3-0d22-4322-85bb-3ff340ecf8a0)

## Dependencies

It was written in _Modern_ C++, compiled to optimized native code, no memory leaks, depends on PostgreSQL native client API (libpq5) and a JSON configuration file (/etc/cppserver/config.json), designed to be run as a container on Kubernetes, from Laptops to the Cloud, or as a native Linux service, it is in fact, a native Linux application, tied to the EPOLL API, it's an async, non-blocking, event-based socket server, it can serve thousands of concurrent connections with only one thread, no [c10K connection problem](https://en.wikipedia.org/wiki/C10k_problem) here.

![basic-dependencies](https://github.com/cppservergit/cppserver-docs/assets/126841556/cfa3cbc6-190c-4baf-859c-f54985efe278)

Thus CPPServer can be defined as a high-peformance, compact, no-code JSON API engine, you don't have to write C++ to create an HTTP API that retuns JSON from SQL queries, it's already written inside CPPServer, you only have to configure your API in config.json and let CPPServer parse this file and serve the requests. Security, observability, tracing, audit logs, native access to the database, it's all included in this program. CPPServer is [open-source](https://github.com/cppservergit/cppserver-pgsql), so it's up to you to add/change features in the code.

It's compiled with GCC 12.1 on Ubuntu 22.04 using C++20 standard.

## Internal structure

The main module starts the server and the pool of worker threads, there is only 1 thread (the main thread, the producer) to manage the socket server using epoll, the pool of workers (consumers) will receive tasks dispatched by the main thread and run the intended service (database I/O), assemble a JSON response and inform EPOLL that we are ready to write, meanwhile, the main thread, the server, can keep processing new connections and reading requests. This is a One-Producer, Many-Consumers model, the code was insired by Bjarne Stroustrup's book "A Tour of C++ 3rd edition" producer-consumer example, section 18.4 "Waiting for events", and also on section 18.5.4 about the use of jthread and stop_token. With the help of Modern C++ we managed to create a simple code structure around the complexity of EPOLL and at the same time obtain benefits from its low-level features.

![thread-pool](https://github.com/cppservergit/cppserver-docs/assets/126841556/e7f13be4-6fc9-446c-a27f-d11dea2349d3)

We keep locking to a minimum, only to coordinate access to the message queue (std::queue) between the Producer and the Consumers, the rest of the program is lock-free, each worker thread has its own database connection, there is no need of connection pooling this way, and the database component is able to recover from an invalid connection in a way transparent to the client, although detailed logs will be recorded when this happens.

The lock-free, controversial part of this program is a Map that contains the socket's FD associated with a data structure that contains the HTTP request (represented as fields, headers, body, etc) and the response buffer, so when a connection is established, a new entry in this map will be created if necessary (it may already exist, in that case no creation overhead), all this happens in the main thread and no other thread will be touching that particular pair of FD and Request/Response object, the FD, which is passed in all EPOLL events is the index to retrieve this high-level Request object, and it is passed (as a reference) to the message queue to be consumed by the worker threads. The thing is, there won't be more than a single thread reading or writing on the same Request/Response object at the same time, there is no data race, but if you use compiler instrumentation or valgrind to test for thread sanity, a lot of warnings will be issued, false positives, because these tools detect that different threads did reads/writes on the same objects without locks. Because the way the program is written, there is no chance for 2 or more threads to perform reads or writes at the same time on the same request object, therefore, no data races in the strict sense, this lock-free management of these requests objects yields high performance, if scoped locks were applied, there would be no warnings on thread sanity, but performance would suffer under high concurrency, A LOT, it was tested and verified.

So, controversial as it may be the use of this Map (std::map) and the passing of the objects's references avoiding a copy to the consumers, it suits the somewhat particular or tricky epoll model, and it was coded in such a way that only one thread can operate at a given time on the same request/response object. There was a sort of natural fit between the use of EPOLL events passing the socket's FD, the Map that associated the FD with a request/response object, and the way non-blocking sockets operate on Linux, and all of this avoiding the data race despite the warnings. The Map is only used by the main thread, there is no need of locks for its operation.

### Single-thread EPOLL manager

Only the main thread, the one that controls the EPOLL event loop, will accept connections and perform read/write operations on sockets, the worker threads only use the assembled request to execute the service using the inputs and produce a JSON output, when the output is ready, the main thread will be notified by EPOLL to write to the socket, all socket operations are non-blocking. The code follow Linux AAPI guidance in order to minimize system calls, for instance, when accepting new connections, a single call `accept4()` accepts and sets the new socket in non-blocking mode:

```
 		  else if (listen_fd == events[i].data.fd) // new connection.
		  {
				struct sockaddr addr;
				socklen_t len;
				len = sizeof addr;
				int fd { accept4(listen_fd, &addr, &len, SOCK_NONBLOCK) };
				if (fd == -1) {
					logger::log("epoll", "error", "connection accept FAILED for epoll FD: " + std::to_string(epoll_fd) + " " + std::string(strerror(errno)));
					continue;
				}

				int rcvbuf {131072}, sndbuf {262144};
				setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof rcvbuf);
				setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &sndbuf, sizeof sndbuf);
				
				if (!buffers.contains(fd))
					 buffers.insert({fd, http::request()});
				else {
					 http::request& req = buffers[fd];
					 if (!req.is_clear) {
						logger::log("epoll", "warn", "invoking request.clear() on FD: " + std::to_string(fd) + " path: " + req.path);
						req.clear();
					 }
				}
								
				mse::update_connections(1);
				epoll_event event;
				event.data.fd = fd;
				event.events = EPOLLIN | EPOLLET | EPOLLRDHUP;
				epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd, &event);
			}
```

### The Producer

When a "read" event arrives, the main thread reads data from the socket while available, assembling the request in parts (fields, headers, etc), when the request is complete the task is dispatched to any of the worker threads, using a queue to store it, the task contains the epoll_fd, the socket FD, and a reference to the specific request object associated with this socket's FD, this is the async part, the control returns inmediately to the main thread while some worker thread is processing in the background the service using the microservice engine module msp.cpp (security checks, database I/O, response JSON assembly, error handling, etc). The "task producer" is shown at the end of the code below:

```
				int fd {events[i].data.fd};
				http::request& req = buffers[fd];
				if (events[i].events & EPOLLIN) {
					bool run_task {false};
					int count {0};
					while ((count = read(fd, data, sizeof(data)))) {
						if (count == -1 && errno == EAGAIN) {
							break;
						}
						if (count > 0) {
							if (read_request(fd, req, data, count)) {
								run_task = true;
								break;
							}
						}
						else {
							logger::log("epoll", "error", "read error FD: " + std::to_string(fd) + " " + std::string(strerror(errno)));
							buffers[fd].clear();
							break;
						}
					}
					if (run_task) {
						//producer
						worker_params wp {epoll_fd, fd, req};
						{
							std::scoped_lock lock {m_mutex};
							m_queue.push(wp);
							m_cond.notify_all();
						}
					}
```

We avoid by all means calling `read()` or `write()` if there is no data available/socket ready, it's one fundamental technique when using non-blocking sockets with epoll, wait for the proper events and signals, don't call the system if it's not 100% necessary.

This is the function call graph for main.cpp which is the module that starts the process and controls the epoll loop:

![main-call-graph](https://github.com/cppservergit/cppserver-docs/assets/126841556/7d0668c8-81a1-4407-a2cf-35af5216cae7)

### The Consumer

The consumer, that is the background thread that processes the task (database I/O mostly) is this function, placed at the bottom of the diagram shown above:

```
void consumer(std::stop_token tok) noexcept 
{
	//start microservice engine on this thread
	mse::init();
	
	while(!tok.stop_requested())
	{
		//prepare lock
		std::unique_lock lock{m_mutex}; 
		//release lock, reaquire it if conditions met
		m_cond.wait(lock, [&tok] { return (!m_queue.empty() || tok.stop_requested()); }); 
		
		//stop requested?
		if (tok.stop_requested()) { lock.unlock(); break; }
		
		//get task
		auto params = m_queue.front();
		m_queue.pop();
		lock.unlock();
		
		//---processing task (run microservice)
		mse::http_server(params.fd, params.req);
		
		//request ready, set epoll fd for output
		epoll_event event;
		event.data.fd = params.fd;
		event.events = EPOLLOUT | EPOLLET | EPOLLRDHUP;
		epoll_ctl(params.epoll_fd, EPOLL_CTL_MOD, params.fd, &event);
	}
	
	//ending task - free resources if necessary
	logger::log("pool", "info", "stopping worker thread", true);
}
```

This function sets the socket FD ready for writing (in EPOLL terms) after the task is completed, that's a crucial part, because it will make the Linux kernel to trigger events to EPOLL for this socket and the code for writing will be executed, thus CPPServer achieves an event-based, async and non-blocking request processing mechanism with a single thread backed by a pool of four (default, it's configurable) worker threads. Using EPOLL properly is a bit tricky, requires a very clearly organized state machine, but it is a very simple API at the same time, a design decision was made to call it directly from C++ and avoid the overhead and complexity of existing C++ network libraries, and considering this is a Linux-specific program, portability was not an issue.

## Signal handling

A program that is going to run as a container or service must respond to STOP signals in order to terminate gracefully, freeing all resources, in the case of CPPServer, following C++ [RAII](https://en.cppreference.com/w/cpp/language/raii) guidelines, CPPServer guarantees that all resources are released, threads stopped, database connections closed, etc.

```
inline int get_signalfd() noexcept 
{
	signal(SIGPIPE, SIG_IGN);
	sigset_t sigset;
	sigemptyset(&sigset);
	sigaddset(&sigset, SIGINT);
	sigaddset(&sigset, SIGTERM);
	sigaddset(&sigset, SIGQUIT);
	sigprocmask(SIG_BLOCK, &sigset, NULL);
	int sfd { signalfd(-1, &sigset, 0) };
	return sfd;
}
```
__Note__: protection against SIGPIPE is also incorporated in the signal handler, which is very important for TCP/IP socket servers.

To achieve RAII, core C++ guidelines must be followed, but also important, OS signals must be properly intercepted too, CPPServer uses Linux-specific APIs for this (function shown above), and the resulting Signal FD (file descriptor) is incorporated into the _objects of interest_ of EPOLL, this way we can be notified by EPOLL events when a STOP signal was sent to the process.

```
	epoll_event event_signal;
	event_signal.data.fd = m_signal;
	event_signal.events = EPOLLIN;
	epoll_ctl(epoll_fd, EPOLL_CTL_ADD, m_signal, &event_signal);
```

Handling the STOP signal with EPOLL:
```
			else if (m_signal == events[i].data.fd) //shutdown
			{
				logger::log("signal", "info", "stop signal received for epoll FD: " + std::to_string(epoll_fd) + " SFD: " + std::to_string(m_signal));
				exit_loop = true;
				break;
			}
```

This will break the EPOLL loop, close server sockets and trigger the order for all threads to stop and release their resources, with some very _Modern C++_ code.
```
	//shutdown workers
	for (auto s: stops) {
		s.request_stop();
		{
			std::scoped_lock lock {m_mutex};
			m_cond.notify_all();
		}
	}
```

## Modular organization

CPPServer uses a classic C-style module organization for its translation units, every module declares its own namespace to enclose all elements (variables, functions, etc), a  header (.H) file to declare the interface and a corresponding .CPP file containing the implementation of the interface, only the elements declared in the interface file will be available to the clients of that module, whatever is not declared in the interface, won't be visible from the implementation module. This translation units are compiled separately, and when using a Makefile, only what has been changed needs to be recompiled (and any targets that depend on it). No header-only libraries are used, except for the JSON parser, which is a 3rd party open-source component. This helps make the compilation process simple and faster, -O3 and link-time-optimization are used for all the targets.

![module-interface](https://github.com/cppservergit/cppserver-docs/assets/126841556/c21ae584-b78f-4868-b182-fbf101da22cb)

[C++20 modules](https://en.cppreference.com/w/cpp/language/modules) are superior to the classic C-style .H/.CPP modules, but sadly in GCC-12.x the C++ 20 Modules implementation is not production-ready yet.

This is the modular structure of CPPServer:

![module-structure](https://github.com/cppservergit/cppserver-docs/assets/126841556/ea5740c8-b056-412d-a487-ce0eb7235938)

The client of a module only has access to the module's interface, implementation details can change and the client won't be affected, this is the case with the module sql.cpp, there is an implementation (default one) for PostgreSQL native API and another for ODBC, which is the native API for SQLServer. The client of this module (mse.cpp) does not get affected by the implementation sql.cpp, the code is clean, each module exposes its own namespace. Below is the implementation of a generic function that executes an SQL command that returns a resultset as a JSON array, whatever the implementation of sql:: is, this code remains the same.

```
	//returns a single resultset
	void dbget(std::string& jsonBuffer, config::microService& ms) 
	{
		sql::get_json(ms.db, jsonBuffer, ms.reqParams.sql(ms.sql, t_user_info.userLogin));
	}
```

### Modules inventory

|Module|Namespace|Description|
|-----------|-----------|-----------|
|main||Main module, starts the server and controls EPOLL loop|
|env|env::|Reads environment variables|
|logger|logger::|prints log messages to stderr in JSON format for LOKI, also can push log record to LOKI if configured (optional)|
|config|config::|Parse config.json, provides microservice and requestParameters structs|
|httputils|http::|Provides abstractions for http request and response|
|loki|loki::|Provides push API via plain http to LOKI server, optional use, may be removed in future versions|
|mse|mse::|Microservice engine, provides the generic functions to execute different types of common microservices, plus the mechanism for invokation given the configuration|
|sql|sql::|Encapsulates the PostgreSQL native API and builds JSON from resultsets when required, a version of this module for ODBC is also available|
|session|session::|Provides functions for security session management using PostgreSQL native API|
|login|login::|Provides login service using PostgreSQL native API|
|audit|audit::|Saves audit record, default implementation uses logger module|

There is fair separation of concerns between these modules, which is good for design, because it helps isolate changes when necessary, let's say there is a bug parsing HTTP request headers, the fix would be isolated to httputils.cpp module implementation, from the build system perspective (Makefile) only this module would require recompilation, and its dependant targets if any.

## Memory management

Great care was put in CPPServer's memory handling, starting with the use of __Modern C++__ abstractions that provide automatic resource management thanks to RAII, by avoiding the use of C-style low code whenever possible and absolutely not using dynamic allocation of memory with low level C/C++ functions.
The present version has been verified free of memory leaks using tools like valgrind and GCC instrumentation, but it was also a design goal to avoid repeated allocation/destruction of objects, which in a server can occur at a high rate leading to memory fragmentation and also imposing lots of work on the C++ allocators, to achieve this we pass by reference when it is convenient to avoid a copy, but we also pre-allocate main objects once, like a thread_local string to store JSON responses, each worker thread has its own thread-safe, memory-safe buffer to store the current JSON response, no allocations will be made for each request, just one, for a server running 7x24 that means some huge savings in memory allocation tasks, the same happens with the request/response objects, the main thread maintains a map that associated a socket FD with the request/response object, this object gets passed by reference to the worker threads. The server can process millions of requests, but it won't be allocating the main buffers (json response, http request/response) millions of times, just once.

Each worker thread manages a copy of the service map (std::unordered_map), which is a representation of the parsed config.json file, this map contains pointers the the microservice functions and validator functions associated with each URI (service API), as defined in config.json, and when a request gets dispatched to a worker thread, it will lookup on this service map using the path/uri of the service requested, if found, the pointers are retrieved, the security checks and inputs validated and if everything is OK the database I/O function will be executed.

![cppserver-configmap](https://github.com/cppservergit/cppserver-docs/assets/126841556/6ea35d75-3aea-419b-bda0-5813fe1a1314)

We use a thread_local variable in the microservice engine module (mse.cpp) to make it thread-safe, memory-safe and fast all at the same time. This is the core of this module, the engine that runs a microservice or JSON API, so to speak:

```
	struct service_engine {
	  public:
		service_engine() {
			m_json_buffer.reserve(32767);
			m_service_map = config::get_config_map();
			for (auto& m: m_service_map) 
			{
				m.second.serviceFunction = getFunctionPointer(m.second.func_service);
				if (!m.second.func_validator.empty())
					m.second.customValidator = getValidatorFunctionPointer(m.second.func_validator);
			}
		}

		inline std::string& run(http::request& req) 
		{
			if (auto m = m_service_map.find( req.path ); m != m_service_map.end() ) {
				if ( m->second.secure ) {
					if ( !sessionUpdate() )
						throw LoginRequiredException();
				}
				m_json_buffer.clear();
				m_json_buffer.append( validateInputs( req.path, req.params, m->second ) );
				if (m_json_buffer.empty() ) {
					m->second.serviceFunction( m_json_buffer, m->second );
					if ( m->second.audit_enabled )
						audit::save(req.path, t_user_info.userLogin, req.remote_ip, m->second);
				}
				return m_json_buffer;
			} else {
				throw std::runtime_error("microservice path not found");
			}
		}

	  private:
		std::unordered_map<std::string, config::microService> m_service_map;
		std::string m_json_buffer;
		
	}; 
	thread_local service_engine t_service;
```

The engine is thread_local, each worker thread has one, and the engine mantains its own string buffer for JSON responses, preallocated at 32K.

With these design choices CPPServer has proven itself capable of processing successfully thousands (7000+) of real microservice requests per second on computers that fit on the palm of the hand, including the overhead of security and inputs checks, as well as database I/O, a balance between performance, code simplicity and stability was sought and to a certain extent it has been achieved in the current version. Please note that when running tests using "echo" services and utilities like Apache Utils (ab) the request-per-second rate is much higher, but the regular stress tests are run using a custom-made, configurable C++ HTTPS client that "impersonates" real users, creates thousands of real security sessions and this way the whole program is put under load, up to 20000 concurrent users have been used in these tests. Keep in mind that on a production environment there will always be an Ingress/Load Balancer in front of CPPServer.

## Database I/O

The generic functions that can be used for a JSON API reside in mse.cpp, these can be services or validators, all of them have the same interface and rely on the sql:: module to perform its work. These functions are invoked by the service_engine::run() function shown above in the previous section. 
Here is an example of some of the most basic functions:
```
	//returns a single resultset
	void dbget(std::string& jsonBuffer, config::microService& ms) 
	{
		sql::get_json(ms.db, jsonBuffer, ms.reqParams.sql(ms.sql, t_user_info.userLogin));
	}

	//returns multiple resultsets from a single query
	void dbgetm(std::string& jsonBuffer, config::microService& ms) {
		sql::get_json(ms.db, jsonBuffer, ms.reqParams.sql(ms.sql, t_user_info.userLogin), ms.varNames);
	}

	//execute data modification query (insert, update, delete) with no resultset returned
	void dbexec(std::string& jsonBuffer, config::microService& ms) {

		constexpr char STATUS_OK[] = "{\"status\": \"OK\"}";
		constexpr char STATUS_ERROR[] = "{\"status\": \"ERROR\",\"description\" : \"System error\"}";

		if (sql::exec_sql(ms.db, ms.reqParams.sql(ms.sql, t_user_info.userLogin)))
			jsonBuffer.append(STATUS_OK);
		else
			jsonBuffer.append(STATUS_ERROR);
	}
```

They receive the worker thread's JSON buffer to store the response, and a variable representing the microservice in question, this is a struct with several fields and functions that automate generating properly formatted SQL without risks of SQL injection attacks, the sql:: module takes care of the hard work, and will trigger an exception in there is a fatal error, that exception will be trapped by the function that wraps/coordinates all the tasks inside the mse:: module, microservice(), and in that case a JSON with the proper ERROR status will be returned, and detailed logs will be printed on the server using the logger:: module, which will use STDERR and/or LOKI (a very popular logs aggregator)

The microservice data structure is central to this mechanism, when config.json is parsed, each entry (if valid) will be converted into a variable of this type:

```
	struct microService 
	{
		std::string db;
		std::string sql;
		bool secure {true};
		requestParameters reqParams;
		std::vector<std::string> varNames; //array names when returning multiple arrays
		std::vector<std::string> roleNames; //authorized roles
		struct validator {
			std::string sql;
			std::string id;
			std::string description;
		} validatorConfig;
		std::string func_service;
		std::string func_validator;
		std::function<void(std::string& jsonResp, microService&)> serviceFunction;
		std::function<void(std::string& jsonResp, microService&)> customValidator;
		bool audit_enabled {false};
		std::string audit_record;
	};
```

A database I/O service will only be invoked if the security checks passed and the inputs checks also passed, otherwise there is no way the code engine will invoke them, that's the contract. The inputs check includes invoking a validator function if any was defined for the service in config.json, validator functions are very similar to service functions, except that these will add a JSON with INVALID status if the validation fails, these are the generic validation functions in the current version:

```
	//if the resultset is not empty then fail (jsonResp will contain something)
	void db_nomatch(std::string &jsonResp, config::microService& ms) {

		const std::string STATUS_ERROR = R"(
		{
			"status": "INVALID",
			"validation": {
				"id": "$id",
				"description": "$description"
			}
		}
		)";

		if (sql::has_rows(ms.db, ms.reqParams.sql(ms.validatorConfig.sql)))
			jsonResp = replaceParam( STATUS_ERROR, { "$id", "$description" }, { ms.validatorConfig.id, ms.validatorConfig.description } );

	}

	//if the resultset is empty then fail (jsonResp will contain something)
	void db_match(std::string &jsonResp, config::microService& ms) {

		const std::string STATUS_ERROR = R"(
		{
			"status": "INVALID",
			"validation": {
				"id": "$id",
				"description": "$description"
			}
		}
		)";

		if (!sql::has_rows(ms.db, ms.reqParams.sql(ms.validatorConfig.sql)))
			jsonResp = replaceParam( STATUS_ERROR, { "$id", "$description" }, { ms.validatorConfig.id, ms.validatorConfig.description } );

	}
```

### JSON declarations and its relation with C++ code

As an API builder using CPPServer, you don't have to write C++, it's already written as shown above, you just have to declare your API in config.json, example:
```
		{
			"db": "db1",
			"uri": "/ms/gasto/update",
			"sql": "call sp_gasto_update($gasto_id,  $fecha, $categ_id, $monto, $motivo)",
			"function": "dbexec",
			"fields": [
				{"name": "gasto_id", "type": "int", "required": "true"},
				{"name": "monto", "type": "double", "required": "true"},
				{"name": "fecha", "type": "date", "required": "true"},
				{"name": "motivo", "type": "string", 	"required": "true"},
				{"name": "categ_id", "type": "int", "required": "true"}
			],
			"roles": [{"name":"can_update"}]
		}
```

Relevant aspects of this JSON record:

* The `db` attribute references an environment variable `CPP_DB1` that contains the connection string, it's an index to lookup an already established database connection using that connection string, you can target several databases from the same config.json.
* The `sql` attribute is the SQL command to be executed, it may contain placeholders to replace pre-validated input values, the mechanism that performs that task already takes care of SQL injection and proper formatting.
* The `function` attribute, that's the key part, it must match one of the service API functions contained in the module mse.cpp, using that string a pointer to the function will be retrieved.

When the program starts, config.json gets parsed and every record is converted into a variable of type config::microservice (shown above in previous section) and stored in an unordered map, indexed by the path (uri) of the service.
This map of microservice objects is the link between the request URI and the function that gets executed in mse.cpp. The are more aspects to highlight about config.json, but it has its own document for that purpose, what is relevant for this overview is to show the relation between the configuration and the execution of the code that does the database I/O. CPPServer uses std::function to invoke the service API function.

### Inventory of service API functions

These are the utility functions that cover a wide variety of business cases, assuming there is a layer of stored procedures/functions in the target database that encapsulate the business logic, as is the case in many corporate SQL databases. There are also some functions for security, diagnostics, monitoring, health check and blobs support.

|Function|Description|
|--------|-----------|
|dbget|Retrieves a single resultset as JSON|
|dbgetm|Retrieves multiple resultsets as JSON|
|dbexec|Executes a data modification query, returns JSON with status OK, don't expects a resultset|
|login|Executes the cpp_dblogin SQL function, if resultset is not empty assumes successful login and creates a security session, uses login:: and session:: modules|
|logout|Executes the cpp_session_delete SQL procedure, always returns JSON with status OK, uses session:: module|
|downloadFile|retrieves a BLOB given its ID from /var/blobs and stores it in the response buffer, for this service the response is not JSON but a file attachment for the browser|
|deleteFile|deletes a BLOB from /var/blobs given its ID and also the corresponding record in the database, returns JSON with status OK|
|getSessionCount|Executes the cpp_session_count SQL function to retrieve a resultset with the number of active sessions, uses session:: module|
|getServerInfo|Returns JSON with relevant metrics of CPPServer, for diagnostics and monitoring|
|getMetrics|Returns the same as above plus getSessionCount() in Prometheus-compatible format for collecting metrics|
|ping|Returns minimal JSON with status OK, it is the intended service to be called for health/liveness checks on Kubernetes or Cloud container services|
|get_version|Returns JSON with POD name and CPPServer version and build date|

__Validation functions__

|Function|Description|
|--------|-----------|
|db_match|Fails if the resultset is empty, if data is not found for the given SQL then validation fails|
|db_nomatch|Fails if the resultset is not empty, if data is found for the given SQL then validation fails|



