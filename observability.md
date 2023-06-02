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
#### CPPServer microservice for metrics

CPPServer has a built-in APIs to obtain metrics in JSON format, in case you are not using Prometheus for monitoring CPPServer:

curl -s http://k8s.mshome.net/ms/status | jq

```
{
  "status": "OK",
  "data": [
    {
      "pod": "cppserver-deployment-67d6df4b57-xwxtc",
      "totalRequests": 292646,
      "avgTimePerRequest": 0.00143762,
      "startedOn": "2023-02-12 22:36:42",
      "connections": 2,
      "activeThreads": 1
    }
  ]
}
```

curl -s http://k8s.mshome.net/ms/sessions | jq

```
{
  "status": "OK",
  "data": [
    {
      "total": 0
    }
  ]
}
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

### HAProxy metrics

In case you are using HAProxy in front of you K8s cluster, HAProxy also incorporates an endpoint to export metrics to Prometheus, a frontend section must be configured in /etc/haproxy/haproxy.cfg:

```
frontend prometheus
   bind *:8405
   http-request use-service prometheus-exporter if { path /metrics }
   no log
```

On the Prometheus side a configuration for HAProxy must be added to prometheus.yml.

```
  - job_name: haproxy
    metrics_path: /metrics
    static_configs:
      - targets: ["k8s.mshome.net:8405"]
```

Once Prometheus is collecting (pulling) all these metrics, you can use it as a DataSource to be used in Grafana Console, create dashboards and alerts (with email notifications) based on particular metrics.

### Kubernetes cluster metrics

Prometheus has a side project called [Node Exporter](https://github.com/prometheus/node_exporter) that allows to export metrics of every node on the Kubernetes cluster (OS related metrics), and there are pre-built dashboards to visualize this metrics in Grafana, pretty slick, it is up to you if you want to enable this exporter on your cluster.

![image](https://github.com/cppservergit/cppserver-docs/assets/126841556/b5b74da2-80dc-422e-8562-23cda98a2034)

## Logs

Let's assume there is an external Loki server that is visible to your MicroK8s cluster. In this case you can install Promtail, a Kubernetes-native agent (Daemonset) that runs on every node of your cluster and will collect for you the logs of all the Pods on every node and push them to the Loki server.
Then you can use Loki as a datasource for Grafana and visualize and query the logs using its advanced features, there are also export facilities and more. It's the modern way of managing logs, cluster-wide, independent of the applications running on the cluster, a well-behaved application only has to print its log messages to STDERR, JSON is the preferred format, leaving timestamps out because Kubernetes takes care of that, and Promtail knows that. CPPServer has LOKI-friendly logs, and we provide a configuration for Nginx Ingress so that its logs are also exported in JSON format (affects HTTP access logs only).

Please read [Promtail Kubernetes Installation](https://grafana.com/docs/loki/latest/clients/promtail/installation/#daemonset-recommended) instructions, keep in mind that you have to edit this section of the YAML configMap to point to your LOKI server:

```
    clients:
    - url: https://{YOUR_LOKI_ENDPOINT}/loki/api/v1/push
```

### NGinx Ingress configMap

This is the configMap that gets installed in the QuickStart tutorial in order to add some features to the Ingress Controller:

```
sudo microk8s kubectl apply -f https://cppserver.com/files/ingress-config.yaml
```

ConfigMap:
```
--- 
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-load-balancer-microk8s-conf
  namespace: ingress
data:
  proxy-body-size: "250M"
  use-forwarded-headers: "true"
  upstream-keepalive-connections: "20000"
  skip-access-log-urls: "/ms/ping"
  log-format-escape-json: "true"
  log-format-upstream: '{
      "timestamp": "$time_iso8601",
      "remote_addr": "$remote_addr",
      "method": "$request_method",
      "request": "$request",
      "status": $status,
      "http_referrer": "$http_referer",
      "http_user_agent": "$http_user_agent",
      "request_id": "$req_id",
      "cppsessionid": "$cookie_CPPSESSIONID"
      "upstream_name", "$proxy_upstream_name",
      "upstream_addr": "$upstream_addr",
      "upstream_duration": $upstream_response_time
    }'
```



