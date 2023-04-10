# Adding a NetworkPolicy to protect access to the Pods

![Ingress-Service-Pod architecture](https://cppserver.com/files/ingress-diagram.png)

CPPServer deployment follows a somewhat standard convention, the CPPServer Pods are exposed via a ClusterIP Service, which is visible from any Node in the Cluster, as well as the pods, but not from outside the Kubernetes network, in any case, the Pods can be accessed via plain HTTP using the Service or via their own IP, and this may be considered an attack surface if the attacker gets control of one of the Nodes (host VM), but we can block this plain access using a Kubernetes object called NetworkPolicy, which allows to establish at the Pod level firewall-like rules, so that for instance only the Ingress Pods are enabled to access the Service, this way we protect direct access to the ClusterIP that exposes the Pods to the whole Cluster as well as direct access to the Pods IP.

In order to test NetworkPolicy with MicroK8s we need a high-availability cluster, with at least 3 nodes, this can be easily mounted using a single Multipass Ubuntu Server VM and then provision 3+ LXD containers inside the VM, using these native Linux containers we can create a Kubernetes cluster, in our case we use 4 nodes for the cluster, with LXD running inside the VM, it looks like this:

![MicroK8s in HA mode with LDX](https://cppserver.com/files/microk8s-ha-cluster.png)

In short, when using Microk8s as a single-node cluster, the K8s networking services provided do not support NetworkPolicy objects, in high-availability Calico is used for this purpose and it does support NetworkPolicy.

Our LXD containers look like this on the VM host:
```
lxc list
+--------+---------+-----------------------------+------+-----------+-----------+
|  NAME  |  STATE  |            IPV4             | IPV6 |   TYPE    | SNAPSHOTS |
+--------+---------+-----------------------------+------+-----------+-----------+
| knode1 | RUNNING | 10.130.171.101 (eth0)       |      | CONTAINER | 0         |
|        |         | 10.1.195.128 (vxlan.calico) |      |           |           |
+--------+---------+-----------------------------+------+-----------+-----------+
| knode2 | RUNNING | 10.130.171.130 (eth0)       |      | CONTAINER | 0         |
|        |         | 10.1.69.192 (vxlan.calico)  |      |           |           |
+--------+---------+-----------------------------+------+-----------+-----------+
| knode3 | RUNNING | 10.130.171.225 (eth0)       |      | CONTAINER | 0         |
|        |         | 10.1.176.192 (vxlan.calico) |      |           |           |
+--------+---------+-----------------------------+------+-----------+-----------+
| knode4 | RUNNING | 10.130.171.164 (eth0)       |      | CONTAINER | 0         |
|        |         | 10.1.3.129 (vxlan.calico)   |      |           |           |
+--------+---------+-----------------------------+------+-----------+-----------+
```

if we enter in the terminal of knode1, our choosen master node, we can see kubernetes nodes:
```
microk8s kubectl get nodes -o wide
NAME     STATUS   ROLES    AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
knode4   Ready    <none>   38h   v1.26.3   10.130.171.164   <none>        Ubuntu 22.04.2 LTS   5.15.0-69-generic   containerd://1.6.15
knode2   Ready    <none>   39h   v1.26.3   10.130.171.130   <none>        Ubuntu 22.04.2 LTS   5.15.0-69-generic   containerd://1.6.15
knode3   Ready    <none>   39h   v1.26.3   10.130.171.225   <none>        Ubuntu 22.04.2 LTS   5.15.0-69-generic   containerd://1.6.15
knode1   Ready    <none>   39h   v1.26.3   10.130.171.101   <none>        Ubuntu 22.04.2 LTS   5.15.0-69-generic   containerd://1.6.15
```

From the terminal of knode1 (or any other node), we get the service that exposes the Pods:
```
sudo microk8s kubectl get service -n cppserver
```

Output (IP address may vary in your case, take note):
```
NAME        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
cppserver   ClusterIP   10.152.183.149   <none>        8080/TCP   40h
```

This is the ClusterIP Service that exposes the Pods to the Cluster nodes.
Now access the Pod (container) using the Service's IP address, CURL and plain HTTP:
```
curl http://10.152.183.149:8080/ms/version
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

We can also obtain each CPPServer Pod's IP:
```
microk8s kubectl get pods -n cppserver -o wide
NAME                         READY   STATUS      RESTARTS   AGE     IP             NODE     NOMINATED NODE   READINESS GATES
cppserver-784f868d9f-fttsp   1/1     Running     0          38h     10.1.195.133   knode1   <none>           <none>
cppserver-784f868d9f-wcmhv   1/1     Running     0          38h     10.1.69.195    knode2   <none>           <none>
cppserver-784f868d9f-bnjld   1/1     Running     0          38h     10.1.3.130     knode4   <none>           <none>
cppserver-784f868d9f-p2fzl   1/1     Running     0          38h     10.1.176.195   knode3   <none>           <none>
```

Now we can go straight to any individual Pod, from any Node in the Cluster. Let's invoke from node1 the Pod running on knode4:
```
curl http://10.1.3.130:8080/ms/version
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

We just confirmed that we can access the Pods' Service from inside the Cluster as well as individual Pods via their IP, we cannot reach this IP from outside the Cluster network, for that purpose we use the Ingress, but for security's sake, we better block this direct access to the Pod, only the Ingress should be able to invoke the Service, in case a Cluster's Node becomes compromised.

From knode1's terminal, let's create the Network Policy YAML for CPPServer (using cppserver namespace):
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

Now install this POlicy on the Cluster:
```
sudo microk8s kubectl apply -f netpol.yaml
```

Output:
```
networkpolicy.networking.k8s.io/cppserver created
```

Test again direct access to the service and to the Pod and it won't work, it's blocked.

If we obtain the Node's IP (ip a, interface eth0) we can invoke the Ingress via HTTPS, this is the same IP assigned by LXD to the instance where the node is running:

```
curl https://10.130.171.101:8080/ms/version -k
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

We can also execute this from the Host VM, because it can see LXD cluster, the containers or light VMs that we provisioned in order to deploy our Kubernetes cluster, we can use any of the IPs of the LXD instances, there is a node and an Ingress assigned for each IP of the LXD cluster:

```
lxc list
+--------+---------+-----------------------------+------+-----------+-----------+
|  NAME  |  STATE  |            IPV4             | IPV6 |   TYPE    | SNAPSHOTS |
+--------+---------+-----------------------------+------+-----------+-----------+
| knode1 | RUNNING | 10.130.171.101 (eth0)       |      | CONTAINER | 0         |
|        |         | 10.1.195.128 (vxlan.calico) |      |           |           |
+--------+---------+-----------------------------+------+-----------+-----------+
| knode2 | RUNNING | 10.130.171.130 (eth0)       |      | CONTAINER | 0         |
|        |         | 10.1.69.192 (vxlan.calico)  |      |           |           |
+--------+---------+-----------------------------+------+-----------+-----------+
| knode3 | RUNNING | 10.130.171.225 (eth0)       |      | CONTAINER | 0         |
|        |         | 10.1.176.192 (vxlan.calico) |      |           |           |
+--------+---------+-----------------------------+------+-----------+-----------+
| knode4 | RUNNING | 10.130.171.164 (eth0)       |      | CONTAINER | 0         |
|        |         | 10.1.3.129 (vxlan.calico)   |      |           |           |
+--------+---------+-----------------------------+------+-----------+-----------+
```

Thanks to this NetworkPolicty, we can only reach the Pods via the Ingress, even if we gain control of a Node's terminal.

