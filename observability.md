# Observability with CPPServer, MicroK8s and Grafana

The whole solution proposed for running CPPServer on Kubernetes, can be integrated with the Grafana observability stack, for metrics and logs collection and visualization, helping solve monitoring and diagnostics in a way transparent to the Pods running on Kubernetes.

![observability-k8s](https://github.com/cppservergit/cppserver-docs/assets/126841556/15bfa16c-b193-4132-9c22-2d0e4150c8bb)

You can run the whole Grafana stack on Kubernetes, but you can also run it outside the cluster, in a separate server (VM) dedicated to that purpose, so that the MicroK8s cluster won't be overloaded with running these Grafana servers, except for the agent Promtail, the one that collects logs from all the Pod and all the Nodes of the cluster, and push them to Loki (the log collector).

Grafana is the console used to visualize the datasources provided by Loki (logs, preferably in JSON format) and Prometheus (metrics). Prometheus pulls the metrics from HTTP endpoints, both NGinx Ingress and CPPServer Pods (including LoginServer) provide endpoints to export metrics in Prometheus-compatible format. Even services running outside the cluster, like HAProxy, the external load balancer, can export metrics to Prometheus.

This way you cover your observability needs using an open-source stack that is very popular in the industry.

__Note__: Installation of the Grafana services and HAProxy is outside the scope of this document. The corresponding documentation of these products cover the installation in detail.

## Prometheus metrics

The hostname shown is just for the example, change it before using the curl command.

### CPPServer

```
curl https://k8s.mshome.net/ms/metrics -ks
```

```
# HELP cpp_requests_total The number of HTTP requests processed by this container.
# TYPE cpp_requests_total counter
cpp_requests_total{pod="cppserver-7f89f44f46-n4dtf"} 51684
# HELP cpp_connections Client tcp-ip connections.
# TYPE cpp_connections counter
cpp_connections{pod="cppserver-7f89f44f46-n4dtf"} 1
# HELP cpp_active_threads Active threads.
# TYPE cpp_active_threads counter
cpp_active_threads{pod="cppserver-7f89f44f46-n4dtf"} 1
# HELP cpp_avg_time Average request processing time in milliseconds.
# TYPE cpp_avg_time counter
cpp_avg_time{pod="cppserver-7f89f44f46-n4dtf"} 0.00001583
# HELP sessions Number of logged in users.
# TYPE sessions counter
sessions{pod="cppserver-7f89f44f46-n4dtf"} 0
```

### LoginServer

```
curl https://k8s.mshome.net/loginserver/metrics -ks
```

```
# HELP cpp_requests_total The number of HTTP requests processed by this container.
# TYPE cpp_requests_total counter
cpp_requests_total{pod="loginserver-859b477cf6-nglj4"} 51709
# HELP cpp_connections Client tcp-ip connections.
# TYPE cpp_connections counter
cpp_connections{pod="loginserver-859b477cf6-nglj4"} 1
# HELP cpp_active_threads Active threads.
# TYPE cpp_active_threads counter
cpp_active_threads{pod="loginserver-859b477cf6-nglj4"} 1
# HELP cpp_avg_time Average request processing time in milliseconds.
# TYPE cpp_avg_time counter
cpp_avg_time{pod="loginserver-859b477cf6-nglj4"} 0.00001527
# HELP sessions Number of logged in users.
# TYPE sessions counter
sessions{pod="loginserver-859b477cf6-nglj4"} 0
```

### NGinx Ingress

```
curl http://k8s.mshome.net:10254/metrics -s
```

The output is very long, only a fragment is shown below.
```
# HELP nginx_ingress_controller_success Cumulative number of Ingress controller reload operations
# TYPE nginx_ingress_controller_success counter
nginx_ingress_controller_success{controller_class="k8s.io/ingress-nginx",controller_namespace="ingress",controller_pod="nginx-ingress-microk8s-controller-44gt8"} 1
# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 93.76
# HELP process_max_fds Maximum number of open file descriptors.
# TYPE process_max_fds gauge
process_max_fds 65536
# HELP process_open_fds Number of open file descriptors.
# TYPE process_open_fds gauge
process_open_fds 16
# HELP process_resident_memory_bytes Resident memory size in bytes.
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 4.2729472e+07
```

### Prometheus configuration to consume metrics

Prometheus configuration file prometheus.yml must be configured for collecting (scraping) metrics from servers, under the section "scrape_configs". To collect the metrics shown above we could use:

```
  - job_name: cppserver
    metrics_path: /ms/metrics
    static_configs:
      - targets: ["k8s.mshome.net"]

  - job_name: ingress
    metrics_path: /metrics
    static_configs:
      - targets: ["k8s.mshome.net:10254"]
```

