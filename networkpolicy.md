# -- DRAFT -- instructions not working as expected yet, requires MicroK8s HA Cluster, i won't work on single-node cluster
# Adding a NetworkPolicy to protect access to the Pods

![Ingress-Service-Pod architecture](https://cppserver.com/files/ingress-diagram.png)

CPPServer deployment follows a somewhat standard convention, the Pods are exposed via a ClusterIP Service, which is visible from any Node in the Cluster, but not from outside the Nodes network, in any case, the Pods can be accessed via plain HTTP using the Service, which plays a Proxy/Load Balancer role for them, and this may be considered an attack surface, but we can block this plain access using a Kubernetes object called NetworkPolicy, and establish a rule so that only the Ingress Pods are enabled to access the Service, this way we protect direct access to the ClusterIP that exposes the Pods to the whole Cluster.

Let's test it, enter your CPPServer single-node cluster terminal and execute:

```
sudo microk8s kubectl get service -n cppserver
```

Output (IP address may vary in your case, take note):
```
NAME        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
cppserver   ClusterIP   10.152.183.149   <none>        8080/TCP   40h
```

This is the CLusterIP Service that exposes the Pods to the Cluster nodes.
Now access the Pod (container) using the Service's IP address, CURL and plain HTTP:
```
curl http://10.152.183.149:8080/ms/version -s | jq
```

Output:
```
{
  "status": "OK",
  "data": [
    {
      "pod": "cppserver-674b6775cd-d8fnp",
      "server": "cppserver v1.01-rev129-20230406"
    }
  ]
}
```

We just confirmed that we can access the Pods' Service from inside the Cluster, we cannot reach this IP from outside the Cluster network, for that purpose we use the Ingress, but as told before, we better block this direct access to the Pod, only the Ingress should be able to invoke the Service, in case a Cluster's Node becomes compromised.

Let's create the Network Policy YAML:
```
cat > netpol.yaml << EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cppserver
  namespace: cppserver
spec:
  podSelector:
    matchLabels:
      app: cppserver
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress
EOF
```

In short, this declares that only the Pods from the Ingress will be able to access the Pods from the CPPServer App.

Now install this on the Cluster:
```
sudo microk8s kubectl apply -f netpol.yaml
```

Output:
```
networkpolicy.networking.k8s.io/cppserver created
```

Test again direct access to the service (it may take a few seconds for the NetworkPolicy to take effect):
```
curl http://10.152.183.149:8080/ms/version -s | jq
```

