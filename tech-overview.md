# CPPServer Technical Overview (in progress)

CPPServer is a minimal (~300K) http server whose sole purpose is to serve GET/POST requests that will in turn invoke functions to execute SQL queries and return JSON to the clients, which can be any HTTP client, including browsers, mobile apps, another server, etc. It was designed to be a no-code API engine, meaning that you can declare or configure your APIs using a JSON file, and CPPServer will parse this file on startup and serve the requests according to this configuration.

![basic-workflow](https://github.com/cppservergit/cppserver-docs/assets/126841556/f774bbb3-0d22-4322-85bb-3ff340ecf8a0)

It was written in _Modern_ C++, compiled to optimized native code, no memory leaks, depends on PostgreSQL native client API (libpq5) and a JSON configuration file (/etc/cppserver/config.json), designed to be run as a container on Kubernetes, from Laptops to the Cloud, or as a native Linux service, it is in fact, a native Linux application, tied to the EPOLL API, it's an async, non-blocking, event-based socket server, it can serve thousands of concurrent connections with only one thread, no [c10K connection problem](https://en.wikipedia.org/wiki/C10k_problem) here.

![basic-dependencies](https://github.com/cppservergit/cppserver-docs/assets/126841556/cfa3cbc6-190c-4baf-859c-f54985efe278)

Thus CPPServer can be defined as a high-peformance, compact, no-code JSON API engine, you don't have to write C++ to create an HTTP API that retuns JSON from SQL queries, it's already written inside CPPServer, you only have to configure your API in config.json and let CPPServer parse this file and serve the requests. Security, observability, tracing, audit logs, native access to the database, it's all included in this program. CPPServer is [open-source](https://github.com/cppservergit/cppserver-pgsql), so it's up to you to add/change features in the code.

## Internal structure

The main module starts the server and the pool of worker threads, there is only 1 thread (the main thread, the producer) to manage the socket server using epoll, the pool of workers (consumers) will receive tasks dispatched by the main thread and run the intended service (database I/O), assemble a JSON response and inform EPOLL that we are ready to write, meanwhile, the main thread, the server, can keep processing new connections and reading requests. This is a One-Producer, Many-Consumers model, the code was insired by Bjarne Stroustrup's book "A Tour of C++ 3rd edition" producer-consumer example, section 18.4 "Waiting for events", and also on section 18.5.4 about the use of jthread and stop_token. With the help of Modern C++ we managed to create a simple code structure around the complexity of EPOLL and at the same time obtain benefits from its low-level features.

![thread-pool](https://github.com/cppservergit/cppserver-docs/assets/126841556/e7f13be4-6fc9-446c-a27f-d11dea2349d3)

We keep locking to a minimum, only to coordinate access to the message queue (std::queue) between the Producer and the Consumers, the rest of the program is lock-free, each worker thread has its own database connection, there is no need of connection pooling this way, and the database component is able to recover from an invalid connection in a way transparent to the client, although detailed logs will be recorded when this happens.

The lock-free, controversial part of this program is a Map that contains the socket's FD associated with a data structure that contains the HTTP request (represented as fields, headers, body, etc) and the response buffer, so when a connection is established, a new entry in this map will be created if necessary (it may already exist, in that case no creation overhead), all this happens in the main thread and no other thread will be touching that particular pair of FD and Request/Response object, the FD, which is passed in all EPOLL events is the index to retrieve this high-level Request object, and it is passed (as a reference) to the message queue to be consumed by the worker threads. The thing is, there won't be more than a single thread reading or writing on the same Request/Response object at the same time, there is no data race, but if you use compiler instrumentation or valgrind to test for thread sanity, a lot of warnings will be issued, false positives, because the way the program is written, there is no chance for 2 or more threads to perform reads or writes at the same time on the same request object, therefore, no data races in the strict sense, this lock-free management of these requests objects yields high performance, if scoped locks were applied, there would be no warnings on thread sanity, but performance would suffer under high concurrency, A LOT, it was tested and verified.

So, controversial as it may be the use of this Map (std::map) and the passing of the objects's references avoiding a copy to the consumers, it suits the somewhat particular or tricky epoll model, and it was coded in such a way that only one thread can operate at a given time of the request/response object. There was a sort of natural fit between the use of EPOLL events passing the socket's FD, the Map that associated the FD with a request/response object, and the way non-blocking sockets operate on Linux, and all of this avoiding the data race despite the warnings. The Map is only used by the main thread, there is no need of locks for its operation.

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

When a "read" event arrives, the main thread reads data from the socket while available, assembling the request in parts (fields, headers, etc), when the request is complete the task is dispatched to any of the worker threads, using a queue to store it, the task contains the epoll_fd, the socket FD, and a reference to the specific request object associated with this socket's FD, this is the async part, the control returns inmediately to the main thread while some worker thread is processing the service using the microservice engine module (security checks, database I/O, response JSON assembly, error handling, etc). The "task producer" is shown at the end of the code below:

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
