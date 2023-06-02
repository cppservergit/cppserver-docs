# Observability with CPPServer, MicroK8s and Grafana

The whole solution proposed for running CPPServer on Kubernetes, can be integrated with the Grafana observability stack, for metrics and logs collection and visualization, helping solve monitoring and diagnostics in a way transparent to the Pods running on Kubernetes.

![observability-k8s](https://github.com/cppservergit/cppserver-docs/assets/126841556/15bfa16c-b193-4132-9c22-2d0e4150c8bb)

You can run the whole Grafana stack on Kubernetes, but you can also run it outside the cluster, in a separate server (VM) dedicated to that purpose, so that the MicroK8s cluster won't be overloaded with running these Grafana servers, except for the agent Promtail, the one that collects logs from all the Pod and all the Nodes of the cluster, and push them to Loki (the log collector).

Grafana is the console used to visualize the datasources provided by Loki (logs, preferably in JSON format) and Prometheus (metrics). Prometheus pulls the metrics from HTTP endpoints, both NGinx Ingress and CPPServer Pods (including LoginServer) provide endpoints to export metrics in Prometheus-compatible format. Even services running outside the cluster, like HAProxy, the external load balancer, can export metrics to Prometheus.

This way you cover your observability needs using an open-source stack that is very popular in the industry.

__Note__: Installation of the Grafana services and HAProxy is outside the scope of this document. The corresponding documentation of these products cover the installation in detail.
