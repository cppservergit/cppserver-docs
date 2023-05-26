# CPPServer Technical Overview

CPPServer is a minimal (300K) http server whose sole purpose is to serve GET/POST requests that will in turn invoke functions to execute SQL queries and return JSON to the clients, which can be any HTTP client, browsers, mobile apps, another server, etc. It was designed to be a no-code API engine, meaning that you can declare or configure your APIs using a JSON file, and CPPServer will parse this file on startup and serve the requests according to this configuration.

It was written in _Modern_ C++, compiled to optimized native code, depends on PostgreSQL native client API (libpq5) and a JSON configuration file (/etc/cppserver/config.json), designed to be run as a container on Kubernetes, from Laptops to the Cloud, or as a native Linux service, it is in fact, a native Linux application, using EPOLL, it's an async, non-blocking, event-based socket server, it can serve thousands of concurrent connections with only one thread, no [c10K connection problem](https://en.wikipedia.org/wiki/C10k_problem) here.

Thus CPPServer can be defined as a high-peformance, compact, no-code JSON API engine, you don't have to write C++ to create an HTTP API that retuns JSON from SQL queries, it's already written inside CPPServer, you only have to configure your API in config.json and let CPPServer parse this file and serve the requests. Security, observability, tracing, audit logs, native access to the database, it's all included in this program. CPPServer is [open-source](https://github.com/cppservergit/cppserver-pgsql), so it's up to you to add/change features in the code.





