# Quickstart with CPPServer and Kubernetes (MicroK8s) in 10 minutes

## Introduction

In a few minutes, using your Windows 10 Pro PC or Laptop, you can install and test our free C++ microservices engine, CPPServer, which runs on Kubernetes, On-Premise, or Cloud, it's portable and vendor-independent, we like to use Canonical MicroK8s (Ubuntu), a very popular and stable Kubernetes distro, easy to use, scales from single-node to high-availability with 3+ nodes, suitable for development and production.

We recommend using Multipass in order to provision VMs on Win10 Pro, it's very easy and fast, like having the cloud on your local machine.

[Download Multipass for Windows 10 Pro](https://multipass.run/download/windows)

If you find any problem following this procedure you may create an Issue on this repository to be promptly assisted or contact us via email: cppserver@martincordova.com

We will create 2 Ubuntu Server 22.04 VMs, one for the database, PostgreSQL 15 running on Docker, and the other for the Kubernetes cluster with MicroK8s. Assuming that you have installed Multipass on your computer and that you have 16GB RAM and about 60GB of free disk space. Let's do it!

![Quickstart Kubernetes cluster with MicroK8s](https://cppserver.com/quickstart-cluster.png)

Disclaimer: no warranties and no responsibilities assumed by the author, use at your own risk.

## 1.- Create a VM for the database
Open a command prompt on Win10 and run:

```
multipass launch docker --name demodb
```

Wait a few minutes and the VM will be ready, launch its Linux terminal:

```
multipass shell demodb
```

Run this command, this will execute all we need in order to download the latest PostgreSQL for Docker, create the database server, restore a backup, etc.

```
curl https://cppserver.com/files/install-demodb -Os && sudo chmod 700 install-demodb && sudo ./install-demodb
```

After a successful installation, you can test it by executing this command:

```
sudo docker exec -e PG_PASSWORD=basica pgsql psql -h 127.0.0.1 -U postgres -c "select version();"
```

Output:
```
PostgreSQL 15.2 (Debian 15.2-1.pgdg110+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 10.2.1-6) 10.2.1 20210110, 64-bit
```

Great! we have the database server up and running, loaded with data, Multipass assigned a name demodb.mshome.net to this VM, and it will be visible from the other VM that we are going to create next. You can exit this Linux terminal by executing the command:

```
exit
```

## 2.- Create a VM for running the Kubernetes cluster
Open a command prompt on Win10 and run:

```
multipass launch --name k8s -c 4 -m 4g -d 8g
```

Enter the Linux terminal on this new VM:

```
multipass shell k8s
```

Update the operating system and install some utilities:

```
sudo DEBIAN_FRONTEND=noninteractive apt update && sudo apt upgrade -y && sudo apt install jq tree net-tools -y
```

## 3.- Install MicroK8s

On the same Linux terminal of the K8s VM you just created, execute:
```
sudo snap install microk8s --classic --channel=1.26
```

Test status:
```
sudo microk8s status --wait-ready
```

Install add-ons:
```
sudo microk8s enable dns hostpath-storage ingress metrics-server
```

Test installation status by checking the system Pods (containers):
```
sudo microk8s kubectl get pods -A
```

Wait (about 3 minutes) and repeat until their status is Running or Completed, six Pods:
```
NAMESPACE     NAME                                       READY   STATUS      RESTARTS   AG
kube-system   coredns-6f5f9b5d74-f5qtj                   1/1     Running     0          3h23m
kube-system   calico-node-fp75s                          1/1     Running     0          3h25m
kube-system   calico-kube-controllers-6ddc98c84b-nbpdt   1/1     Running     0          3h25m
kube-system   hostpath-provisioner-69cd9ff5b8-v94rj      1/1     Running     0          3h22m
ingress       nginx-ingress-microk8s-controller-p598s    1/1     Running     0          3h22m
kube-system   metrics-server-6f754f88d-mgh66             1/1     Running     0          3h22mE
```

Now we will add some extra configuration for the Ingress NGinx (the proxy/load balancer in front of our microservices Container), so it can accept uploads up to a certain size, including some fields in the access-log output, pass proxy headers to the backend Pods, etc.
```
sudo microk8s kubectl apply -f https://cppserver.com/files/ingress-config.yaml
```

Congrats! you just finished MicroK8s single-node cluster installation, even with some extras not installed by default, now we will deploy CPPServer on this MicroK8s cluster for development and testing tasks.

## 4.- Deploy CPPServer and its required objects

First, we will create a namespace to store all CPPServer-related objects in the cluster, this way we keep them separated from the rest, which is a good practice.

On the same Linux terminal, execute:
```
sudo microk8s kubectl create namespace cppserver
```

Expected output: 
```
namespace/cppserver created
```

Now create the secrets, these contain the database connection strings and are created as separate objects, so they can be managed by authorized Cluster admins only, applications can be re-deployed whenever these are updated. A Kubernetes secret will be injected into an environment variable in order to be used by the application.

Execute:
```
sudo microk8s kubectl create secret generic cpp-secret-db1 -n cppserver \
  --from-literal=connstr="host=demodb.mshome.net port=5432 dbname=demodb connect_timeout=10 user=postgres password=basica application_name=CPPServer" \
  --dry-run=client -o yaml | sudo microk8s kubectl apply -f -
```

Expected output: 
```
secret/cpp-secret-db1 created
```

Execute:
```
sudo microk8s kubectl create secret generic cpp-secret-sessiondb  -n cppserver \
  --from-literal=connstr="host=demodb.mshome.net port=5432 dbname=demodb connect_timeout=10 user=cppserver password=basica application_name=CPPServer" \
  --dry-run=client -o yaml | sudo microk8s kubectl apply -f -
```

Expected output: 
```
secret/cpp-secret-sessiondb created
```

Execute:
```
sudo microk8s kubectl create secret generic cpp-secret-logindb  -n cppserver \
  --from-literal=connstr="host=demodb.mshome.net port=5432 dbname=demodb connect_timeout=10 user=cppserver password=basica application_name=CPPServer" \
  --dry-run=client -o yaml | sudo microk8s kubectl apply -f -
```

Expected output: 
```
secret/cpp-secret-logindb created
```

List the secrets in the domain:
```
sudo microk8s kubectl get secrets -n cppserver
```

Expected output: 
```
NAME                  TYPE    DATA  AGE
cpp-secret-db1        Opaque  1     99s
cpp-secret-sessiondb  Opaque  1     58s
cpp-secret-logindb    Opaque  1     46s
```

A quick side note: CPPServer uses 3 database connections, the business database, the security session database, and the login database, in this QuickStart case, all point to the same database, but in production, it won't be the case. In a bit more refined setup, CPPServer delegates login tasks to a specialized container, LoginServer, which also provides LDAP login, and in that scenario, only LoginServer will use the login database connection.

Now we download and create a Kubernetes configMap to contain CPPServer's declaration of microservices, a JSON file named config.json, using a Kubernetes volume CPPServer will have access to this file, this is OK (and convenient) for development, on Production this file should be included in the container image.

Execute:
```
curl https://cppserver.com/files/config.json -Os
```

Execute:
```
sudo microk8s kubectl create configmap cppserver-config  -n cppserver --from-file config.json --dry-run=client -o yaml | sudo microk8s kubectl apply -f -
```

Expected output:
```
configmap/cppserver-config created
```

Let's download the Demo website, to be served by CPPServer, and also create www and blobs directories inside your $HOME directory, CPPServer will use Kubernetes volumes mapped to the Host using these directories, which is OK for development on single-node clusters, but not suitable for production when using many nodes (VMs) for the cluster, in that case, we would use volumes shared by all nodes using shared storage, like NFS or a cluster-native solution.

```
mkdir $HOME/blobs -p && mkdir $HOME/www -p && curl https://cppserver.com/files/demo-k8s.tgz -Os && tar xzf demo-k8s.tgz -C $HOME/www && rm demo-k8s.tgz
```

Execute "tree www" and you should see the structure of the Demo website:
```
www
└── demo
    ├── categ.html
    ├── css
    │   ├── frontend.css
    │   ├── highcharts.css
    │   ├── pico.css
    │   └── pico.css.map
    ├── customer.html
    ├── gasto.html
    ├── images
    │   └── apple-icon.png
    ├── index.html
    ├── js
    │   ├── common.js
    │   ├── frontend.js
    │   ├── highcharts-3d.js
    │   ├── highcharts-3d.js.map
    │   ├── highcharts.js
    │   ├── highcharts.js.map
    │   ├── index.js
    │   └── locale.js
    ├── menu.html
    ├── products.html
    ├── sales.html
    ├── shippers.html
    └── upload.html
```

Download and prepare cppserver.yaml, the deployment file to be consumed by Kubernetes in order to deploy the application, this file contains definitions, configuration, environment variables, volumes, etc. 
```
curl https://cppserver.com/files/cppserver-mk8s.yaml -Os && envsubst < cppserver-mk8s.yaml > cppserver.yaml && rm cppserver-mk8s.yaml
```

Execute to deploy CPPServer:
```
sudo microk8s kubectl apply -f cppserver.yaml
```

Expected output:
```
deployment.apps/cppserver created
service/cppserver created
ingress.networking.k8s.io/cppserver created
cronjob.batch/cppjob created
```

Check the Pods (containers):
```
sudo microk8s kubectl get pods -n cppserver
```

Expected output:
```
cppserver-96476d8d5-llpxk   1/1     Running     0          90s
cppjob-27998802-7l9jd       0/1     Completed   0          25s
```

Wait and repeat until the pods are in Running or Completed status.

Let's print Pod's logs, take note of your CPPServer Pod's name, it will be different from the example shown above:

Execute:
```
sudo microk8s kubectl logs -n cppserver cppserver-96476d8d5-llpxk
```

Expected output:
```
{"source":"signal","level":"info","msg":"signal interceptor registered"
{"source":"env","level":"info","msg":"port: 8080"}
{"source":"env","level":"info","msg":"pool size: 4"}
{"source":"env","level":"info","msg":"http log: 0"}
{"source":"env","level":"info","msg":"stderr log: 1"}
{"source":"env","level":"info","msg":"login log: 0"}
{"source":"env","level":"info","msg":"loki push disabled"}
{"source":"server","level":"info","msg":"cppserver-96476d8d5-llpxk PID: 1 starting cppserver v1.01-rev129-20230310"}
{"source":"server","level":"info","msg":"hardware threads: 4 GCC: 12.1.0"}
{"source":"config","level":"info","msg":"parsing /etc/cppserver/config.json"}
{"source":"config","level":"warn","msg":"microservice /ms/customer/info is not secure"}
{"source":"config","level":"warn","msg":"microservice /ms/customer/search is not secure"}
{"source":"config","level":"warn","msg":"microservice /ms/status is not secure"}
{"source":"config","level":"warn","msg":"microservice /ms/metrics is not secure"}
{"source":"config","level":"warn","msg":"microservice /ms/login is not secure"}
{"source":"config","level":"warn","msg":"microservice /ms/sessions is not secure"}
{"source":"config","level":"warn","msg":"microservice /ms/version is not secure"}
{"source":"config","level":"warn","msg":"microservice /ms/ping is not secure"}
{"source":"config","level":"info","msg":"config.json parsed"}
{"source":"mse","level":"info","msg":"starting microservice engine","thread":"140710382466624","x-request-id":""}
{"source":"mse","level":"info","msg":"starting microservice engine","thread":"140710365681216","x-request-id":""}
{"source":"mse","level":"info","msg":"starting microservice engine","thread":"140710374073920","x-request-id":""}
{"source":"mse","level":"info","msg":"starting microservice engine","thread":"140710242154048","x-request-id":""}
{"source":"epoll","level":"info","msg":"starting epoll FD: 5"}
{"source":"epoll","level":"info","msg":"listen socket FD: 9 port: 8080"}}
```

## 5.- Test the microservices

```
curl https://k8s.mshome.net/ms/status -ks | jq
```

```
{ 
 "status": "OK",
  "data": [
    {
      "pod": "cppserver-96476d8d5-llpxk",
      "totalRequests": 12354,
      "avgTimePerRequest": 3.915e-05,
      "startedOn": "2023-03-26 14:35:15",
      "connections": 1,
      "activeThreads": 1
    }
  ]
}
```

```
curl https://k8s.mshome.net/ms/version -ks | jq
```

```
{ 
 "status": "OK",
  "data": [
    {
      "pod": "cppserver-96476d8d5-llpxk",
      "server": "cppserver v1.01-rev129-20230310"
    }
  ]
}
```

```
curl https://k8s.mshome.net/ms/customer/search?filter=a -ks | jq
```

```
{
  "status": "OK",
  "data": [
    {
      "customerid": "ALFKI",
      "companyname": "Alfreds Futterkiste"
    },
    {
      "customerid": "ANATR",
      "companyname": "Ana Trujillo Emparedados y helados"
    },
    {
      "customerid": "ANTON",
      "companyname": "Antonio Moreno Taquería"
    },
    {
      "customerid": "AROUT",
      "companyname": "Around the Horn"
    }
  ]
}
```

```
curl https://k8s.mshome.net/ms/customer/info?customerid=ANATR -ks | jq
```

```
{  
  "status": "OK",
  "data": {
    "customer": [
      {
        "customerid": "ANATR",
        "contactname": "Ana Trujillo",
        "companyname": "Ana Trujillo Emparedados y helados",
        "city": "México D.F.",
        "country": "Mexico",
        "phone": "(5) 555-4729"
      }
    ],
    "orders": [
      {
        "orderid": 10308,
        "orderdate": "1994-10-19",
        "shipcountry": "Mexico",
        "shipper": "Federal Shipping",
        "total": 88.8
      },
      {
        "orderid": 10625,
        "orderdate": "1995-09-08",
        "shipcountry": "Mexico",
        "shipper": "Speedy Express",
        "total": 479.75
      },
      {
        "orderid": 10759,
        "orderdate": "1995-12-29",
        "shipcountry": "Mexico",
        "shipper": "Federal Shipping",
        "total": 320
      },
      {
        "orderid": 10926,
        "orderdate": "1996-04-03",
        "shipcountry": "Mexico",
        "shipper": "Federal Shipping",
        "total": 514.4
      }
    ]
  }
}
```

## 6.- Demo webapp

As an extra-bonus, CPPServer includes a lightweight, web-responsive application framework for those interested in producing fast results on the front-end side, the Demo webapp can be used as a template and starting point, it's independent of the backend, it consumes JSON according to CPPServer's output specification, you can login using: user1, basica

Visit your local webapp: https://k8s.mshome.net/demo and ignore the invalid certificate warning.

![Frontend.js - a free lighweight web-mobile framework](https://cppserver.com/ui.png)

This web development model was built from the start to be easily converted into a mobile native App using Apache Cordova, it takes about 1 hour to create a Mobile App with biometric login and more, based on the very same website you use for desktop browsers, we called this framework "WebR". It's free to use, even for commercial purposes, at your own risk, please read the disclaimer at the beginning of this article.

There are 10.000 users available (user1...user10000), these are used by a stress test client application we use to test CPPServer reliability, more on this later if you are interested in running these tests, contact us! 

Congrats! you just finished the first part of this tutorial, what comes next? a series of typical development tasks, like adding your own microservices to config.json and refreshing the Cluster to read your changes, and more, if you are interested in part 2 please let us know, it's free!

Contact us: cppserver@martincordova.com


