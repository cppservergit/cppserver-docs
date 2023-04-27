# Common administrative tasks

These are step-by-step instructions to execute the most common Admin tasks, but also those tasks that will be frequently required during development, like adding a microservice definition and then restart the Pods of CPPServer.

These tasks assume that you have a working cluster according to the QuickStart tutorial available in this Repo.

It's highly recommended at this point after completing the QuickStart tutorial, to get yourself introduced to basic Kubernetes concepts, there are excellent brief tutorials on the Internet, also check the contents of this Repo and read "Anatomy of a C++ microservice" to get a quick view of CPPServer concepts. Both materials will give you a better understanding of the tasks below.

OK, let's start, enter in your k8s terminal (you created this VM during the QuickStart tutorial):
```
multipass shell k8s
```

## View Ingress access logs

NGinx Ingress exposes your Pods (via a Service) to the external network, it works as an HTTP Proxy, it will record HTTP access logs in its own format, the format and fields is configurable via an Nginx ConfigMap, we already applied this ConfigMap to the Ingress during the QuickStart tutorial.

Execute:
```
sudo microk8s kubectl get pods -n ingress
```

Expected output:
```
NAME                                      READY   STATUS    RESTARTS   AGE
nginx-ingress-microk8s-controller-2b76l   1/1     Running   0          26m
```

We will use the name of the Ingress pod to view its logs, remember to use the pod name of your output:
```
sudo microk8s kubectl logs -n ingress nginx-ingress-microk8s-controller-2b76l
```

Expected output (shown abbreviated here, it may be long):
```
...
[2023-04-20T05:21:25+00:00] 172.20.176.1 "GET /ms/shippers/view HTTP/2.0" 200 456 "https://k8s.mshome.net/demo/shippers.html" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36" 61 0.007 9524-9c6ab8b5-6c4b-4e5f-be30-a4ffcc55dc2e [cppserver-cppserver-8080] 10.1.77.6:8080 456 0.006 200 24b7a5c93d4228e02d5ac1d2ad758b40
[2023-04-20T05:21:26+00:00] 172.20.176.1 "GET /demo/menu.html HTTP/2.0" 200 1005 "https://k8s.mshome.net/demo/shippers.html" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36" 31 0.000 9524-9c6ab8b5-6c4b-4e5f-be30-a4ffcc55dc2e [cppserver-cppserver-8080] 10.1.77.6:8080 1005 0.001 200 5cfe9f0790d146e3dc0e21ed2ef3501e
[2023-04-20T05:21:27+00:00] 172.20.176.1 "GET /demo/sales.html HTTP/2.0" 200 4844 "https://k8s.mshome.net/demo/menu.html" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36" 32 0.001 9524-9c6ab8b5-6c4b-4e5f-be30-a4ffcc55dc2e [cppserver-cppserver-8080] 10.1.77.6:8080 4844 0.001 200 6b0a3890a78bc7b7da87d1d22b16e4b7
[2023-04-20T05:21:27+00:00] 172.20.176.1 "GET /demo/js/highcharts-3d.js HTTP/2.0" 200 49318 "https://k8s.mshome.net/demo/sales.html" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36" 36 0.002 9524-9c6ab8b5-6c4b-4e5f-be30-a4ffcc55dc2e [cppserver-cppserver-8080] 10.1.77.6:8080 49318 0.001 200 d677db4114b83da3cda8da4ec126ed94
...
```

Errors, warnings and informational messages will be recorded in this log output besides the regular http access logs. If there are problems reaching the Pods, this is one of the logs to check.

## Change CPPServer environment variables

In your home directory (/home/ubuntu) which should be your current working directory in the Multipass shell, there is a file named cppserver.yaml, it was used to deploy the CPPServer application, it does contain the definition of the deployment and the configuration of the container, in the form of environment variables, we will change a specific variable to enable LOGIN logs. Familiarity with NANO text editor is required.

Execute:
```
nano cppserver.yaml
```

Scroll down and locate this:
```
          - name: CPP_LOGIN_LOG
            value: "0"
```

Change value to "1", don't alter the indentation. It should look like this:
```
          - name: CPP_LOGIN_LOG
            value: "1"
```
CTRL-x to save and exit to the terminal.

Apply the changes:
```
sudo microk8s kubectl apply -f cppserver.yaml
```

Expected output:
```
deployment.apps/cppserver configured
service/cppserver unchanged
ingress.networking.k8s.io/cppserver unchanged
cronjob.batch/cppjob configured
```

When you change something in the deployment file (cppserver.yaml) you just run the command above to apply changes, if you change some resource defined outside this file, like a secret or a configMap, then you need another command, a rollout, to redeploy the Pods and reload these external resources. More on that later.
In the next section we will check the logs of the CPPServer Pod to verify the new value of the environment variable.

## View CPPServer logs

List CPPServer Pods:

Execute:
```
sudo microk8s kubectl get pods -n cppserver -l app=cppserver
```

Expected output:
```
NAME                         READY   STATUS    RESTARTS   AGE
cppserver-86d7c566f8-56n9p   1/1     Running   0          2m25s
```

To obtain the logs (please use the NAME value of your own output):
```
sudo microk8s kubectl logs -n cppserver cppserver-86d7c566f8-56n9p
```

You should see a fresh CPPServer pods log (it was restarted) reflecting the change you made to the environment variable in the previous task:
```
{"source":"signal","level":"info","msg":"signal interceptor registered"}
{"source":"env","level":"info","msg":"port: 8080"}
{"source":"env","level":"info","msg":"pool size: 4"}
{"source":"env","level":"info","msg":"http log: 0"}
{"source":"env","level":"info","msg":"stderr log: 1"}
{"source":"env","level":"info","msg":"login log: 1"}
{"source":"env","level":"info","msg":"loki push disabled"}
{"source":"server","level":"info","msg":"cppserver-d6b5c6f49-fj5x8 PID: 1 starting cppserver v1.01-rev129-20230406"}
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
{"source":"mse","level":"info","msg":"starting microservice engine","thread":"140347369195072","x-request-id":""}
{"source":"epoll","level":"info","msg":"starting epoll FD: 4"}
{"source":"epoll","level":"info","msg":"listen socket FD: 5 port: 8080"}
{"source":"mse","level":"info","msg":"starting microservice engine","thread":"140347360802368","x-request-id":""}
{"source":"mse","level":"info","msg":"starting microservice engine","thread":"140347352409664","x-request-id":""}
{"source":"mse","level":"info","msg":"starting microservice engine","thread":"140347344016960","x-request-id":""}
```

This is the line that show your change to the configuration:
```
{"source":"env","level":"info","msg":"login log: 1"}
```

To add date/time to the log's output add --timestamps to the command:
```
sudo microk8s kubectl logs -n cppserver cppserver-86d7c566f8-56n9p --timestamps
```

Login into the Demo WebApp (see QuickStart tutorial) and repeat this command to print the logs, you should see a log message regarding your login activity at the end of the logs:
```
{"source":"security","level":"info","msg":"login OK - user: mcordova IP: 172.20.176.1 sessionid: 9527-b72caf20-1224-4dc2-89d5-2a6a9be556e7 roles: can_delete, can_insert, can_update, sysadmin","thread":"140347344016960","x-request-id":"6c31fef563e44b1f89cbc67bfafcfaaf"}
```

CPPServer records errors, warnings and informational messages according to its configuration via environment variables, it follows best practices for logging, like using JSON, it won't include timestamps because that's added by Kubernetes. Errors and warnings will be recorded like these:
```
{"source":"login","level":"warn","msg":"void login::dbutil::reset_connection() connection to database demodb no longer valid, reconnecting... ","thread":"140347352409664","x-request-id":"8b2e9d47966518d7262e4cff529a2011"}
{"source":"login","level":"error","msg":"void login::dbutil::reset_connection() error reconnecting to database demodb - could not translate host name 'demodb.mshome.net' to address: Temporary failure in name resolution ","thread":"140347352409664","x-request-id":"8b2e9d47966518d7262e4cff529a2011"}
{"source":"login","level":"warn","msg":"void login::dbutil::reset_connection() connection to database demodb no longer valid, reconnecting... ","thread":"140347352409664","x-request-id":"8b2e9d47966518d7262e4cff529a2011"}
{"source":"login","level":"info","msg":"void login::dbutil::reset_connection() connection to database demodb restored","thread":"140347352409664","x-request-id":"8b2e9d47966518d7262e4cff529a2011"}
```

## Add a microservice to config.json

Now you are going to edit config.json, declare a microservice definition, save it, update the configMap using this file and restart the deployment so it can see the new version of the configMap, this is a frequent and typical DevOps task when implementing microservices with CPPServer and Kubernetes. The file config.json is stored in your home directory:

```
nano config.json
```

Locate the beginning of the file, place the cursor under this section:
```
{
        "services":
        [
```

Paste this JSON fragment (including the coma at the end!), this is a microservice definition:
```
               {
                        "db": "db1",
                        "uri": "/ms/test",
                        "sql": "select * from sp_shippers_view()",
                        "function": "dbget",
                        "secure": 0
                },
```

It should look like this:
```
{
        "services":
        [
                {
                        "db": "db1",
                        "uri": "/ms/test",
                        "sql": "select * from sp_shippers_view()",
                        "function": "dbget",
                        "secure": 0
                },
                {
                        "db": "db1",
                        "uri": "/ms/gasto/view",
                        "sql": "select * from sp_gasto_view()",
                        "function": "dbget"
                },
...rest of the file...
```

We use "secure": 0 to allow unsecure execution (no previous login required), this is OK for quick testing, CPPServer always prints warnings when a Pod starts to signal all unsecure services in config.json. CTRL-x to save and exit.

Update the configMap:
```
sudo microk8s kubectl create configmap cppserver-config  -n cppserver --from-file config.json --dry-run=client -o yaml | sudo microk8s kubectl apply -f -
```

Expected output:
```
configmap/cppserver-config configured
```

Now we need to restart the deployment so the Pods can read this new configuration:
```
sudo microk8s kubectl rollout restart deployment cppserver -n cppserver
```

Expected output:
```
deployment.apps/cppserver restarted
```

This is a fundamental command, if you change something not defined in cppserver.yaml (but referenced) you must run "rollout" in order to reload the Pods and reload the new resources, it might be a configMap as in this case, secrets or an updated container image that has the same version tag (so no changes in cppserver.yaml were made).

Now let's test your new microservice:
```
curl https://k8s.mshome.net/ms/test -ks | jq
```

Expected output:
```
{
  "status": "OK",
  "data": [
    {
      "shipperid": 503,
      "companyname": "Century 22 Courier",
      "phone": "800-WE-CHARGE"
    },
    {
      "shipperid": 13,
      "companyname": "Federal Courier Venezuela",
      "phone": "555-6728"
    },
    {
      "shipperid": 3,
      "companyname": "Federal Shipping",
      "phone": "(503) 555-9931"
    },
    {
      "shipperid": 1,
      "companyname": "Speedy Express",
      "phone": "(503) 555-9831"
    },
    {
      "shipperid": 2,
      "companyname": "United Package",
      "phone": "(505) 555-3199"
    },
    {
      "shipperid": 501,
      "companyname": "UPS",
      "phone": "500-CALLME"
    }
  ]
}
```

You've just created a C++ microservice without writing or compiling any C++ code, it's running at full machine-code speed thanks to the declarative facilities of CPPServer. You can exercise your skills checking the access logs of the Nginx Ingress to see how much time was spent processing this request.

## Updating the static website

In this QuickStart deployment we use the static website as an external resource, a Kubernetes volume mapped to a host file system directory (/home/ubuntu/www), if you change any of the files under that directory, you just have to do a "rollout restart" in order reload the updated content, because CPPServer internal micro-webserver does use RAM cache to serve the static content at full speed, you won't see your changes to the content if you don't rollout-restart the deployment, take note please.

Edit website and change the title of the page:
```
nano www/demo/index.html
```

Change it to:
```
                <title>Hello World Kubernetes!</title>
```
CTRL-x to save the file.

Restart the Pods to reload the website:
```
sudo microk8s kubectl rollout restart deployment cppserver -n cppserver
```

Expected output:
```
deployment.apps/cppserver restarted
```

Using a browser navigate to the page:
```
https://k8s.mshome.net/demo/index.html
```
You should see the updated page title on the browser Tab. Whenever you change the static content of the website, you must execute a rollout so the Pods will be recreated and will read the files from disk storage the first time these are requested, afterwards the content will be server from the Pods' RAM, this is by-design of the micro-HTTP-server inside CPPServer, this behavour may change in the future and no rollout could be required.
Also consider that the website might be incorporated into the container, in this case, in order to update the website a rollout must be executed to reload the Pods, or if you change the container's version in cppserver.yaml, then a "microk8s kubectl apply -f ...." must be executed so that Kubernetes will pull the new image and recreate the Pods. The website storage strategy is up to you and depends on your deployment architecture (multiple pods across many nodes?) and your specific development needs. For single-node deployment like the one used in this tutorial, having a local filesystem storage mapped to a volume makes sense and allow for easy refresh to see the new content, this can be automated with a script, for production environments with high-availability clusters (3+ nodes), using an NFS server that all Pod instances can access may be a more suitable solution and easy to configure too.

## Managing database connections with Kubernetes secrets

CPPServer relies on Kubernetes secrets that are later injected into environment variables, during the QuickStart tutorial you defined 3 secrets (logindb, sessiondb and db1) to provide in a secure way the database connection properties for each database, this is an administrative task on production environments, programmers should have no access to these objects.

We are not goint to change any secret yet, unless you have your own PostgreSQL database and you want to create microservices using that database.
This is how to define the secret for the business database, db1:

```
sudo microk8s kubectl create secret generic cpp-secret-db1 -n cppserver \
  --from-literal=connstr="host=demodb.mshome.net port=5432 dbname=demodb connect_timeout=10 user=postgres password=basica application_name=CPPServer" \
  --dry-run=client -o yaml | sudo microk8s kubectl apply -f -
```

To create/modify a secret you must have proper access to a node in the cluster, on production environment this should be restricted to Admins only.

In cppserver.yaml we reference this secret using a secure environment variable:
```
          - name: CPP_DB1
            valueFrom:
              secretKeyRef:
                name: cpp-secret-db1
                key: connstr
                optional: false
```

If you change a secret, a rollout is required in order to use the new version of the secret:
```
sudo microk8s kubectl rollout restart deployment cppserver -n cppserver
```

There is a version of CPPServer for Microsoft SQLServer, it's a different container image, [cppserver/mssql](https://hub.docker.com/r/cppserver/mssql) that must be referenced in cppserver.yaml instead of the default PostgreSQL version [cppserver/pgsql](https://hub.docker.com/r/cppserver/pgsql), for both variants of the container the SessionDB will always be a PostgreSQL DB, but the business database (db1...dbN) must be defined according to the container used. This is an example of the same cpp-secret-db1 for SQL Server using an open source ODBC driver for Linux:
```
sudo microk8s kubectl create secret generic cpp-secret-db1 -n cppserver \
  --from-literal=connstr="Driver=FreeTDS;SERVER=demodb.mshome.net;PORT=1433;DATABASE=demodb;UID=sa;PWD=basica;APP=CPPServer;Encryption=off;ClientCharset=UTF-8" \
  --dry-run=client -o yaml | sudo microk8s kubectl apply -f -
```

If you want to use CPPServer for SQLServer then you have to change in this order:
* Update cpp-secret-db1 using the appropiate connection properties for your case, as shown above.
* Edit cppserver.yaml and change the image property to reference cppserver/mssql instead of cppserver/pgsql
* Use the command to apply cppserver.yaml, no need to rollout (sudo microk8s kubectl apply -f cppserver.yaml).

## Deploy multiple Pods

Edit cppserver.yaml and locate this line:
```
spec:
  replicas: 1
```

Change it to:
```
spec:
  replicas: 2
```
CTRL-x to save the file.

Restart the Pods to reload the website:
```
sudo microk8s kubectl apply -f cppserver.yaml
```

Expected output:
```
deployment.apps/cppserver configured
service/cppserver unchanged
ingress.networking.k8s.io/cppserver unchanged
cronjob.batch/cppjob configured
```

Test it:
```
sudo microk8s kubectl get pods -n cppserver -l app=cppserver
```

Expected output:
```
NAME                        READY   STATUS    RESTARTS   AGE
cppserver-97bcd75c7-9wchn   1/1     Running   0          85m
cppserver-97bcd75c7-zqsms   1/1     Running   0          77s
```

Whenever a request is made to a microservice, the Ingress will load balance the request between these two Pods, using the Service defined between the Pods and the Ingress, please inspect the file cppserver.yaml to see the definition of each object, the Pod template, the Service and the Ingress, the later exposes on the Node (VM) IP address the microservices via HTTP/HTTPS (with a self-generated certificate).

If you were using a multi-node (multiple VMs) cluster, the Pods would be instantiated on any node of the cluster, depending on several factors, Kubernetes takes care of all that. In that case you can add a flag to the command to list the pods, so you can see on which node are they running:
```
sudo microk8s kubectl get pods -n cppserver -l app=cppserver -o wide
```

In our single-node case, the node is the same for both Pods:
```
NAME                        READY   STATUS    RESTARTS   AGE     IP           NODE   NOMINATED NODE   READINESS GATES
cppserver-97bcd75c7-9wchn   1/1     Running   0          92m     10.1.77.46   k8s    <none>           <none>
cppserver-97bcd75c7-zqsms   1/1     Running   0          7m55s   10.1.77.21   k8s    <none>           <none>
```

## Print logs of all the Pods in one namespace

If you have more than one Pod, printing logs Pod-by-Pod is tedious, with this script you can print the logs of all Pods at once.
Just paste this bash script on your terminal and press enter.

### NGinx Ingress

```
for pod in $(sudo microk8s kubectl get pods -o name -n ingress); 
  do sudo microk8s kubectl logs ${pod} -n ingress
done
```

### CPPServer

```
for pod in $(sudo microk8s kubectl get pods -o name -n cppserver); 
  do sudo microk8s kubectl logs ${pod} -n cppserver --timestamps
done
```

This works for any number of Pods and Nodes, it's cluster-wide.

## Testing endpoints for Prometheus metrics

CPPServer, Nginx Ingress Controller and the Kubernet system itself, all of them can export metrics for Prometheus, the most popular open-source metrics collector system, by Grafana. This is very important for the observability of your cluster, and it is a common practice in modern Cloud platforms. Both CPPServer and Ingress have built-in support for this, for the MicroK8s Cluster some additional components are required.

CPPServer metrics endpoint:
```
curl http://k8s.mshome.net/ms/metrics
```

Ingress metrics endpoint:
```
curl http://k8s.mshome.net:10254/metrics
```

For the cluster nodes to export metrics to Prometheus some components must be installed on the cluster, this is covered in this article:
[Kubernetes Node exporter](https://devopscube.com/node-exporter-kubernetes/)

It will allow every node of the cluster to collect OS related metrics and export an endpoint so that Prometheus can collect metrics of the whole cluster, Prometheus can run as a standalone server outside the cluster, or as an application deployed inside the Kubernetes cluster, which makes this whole setup more tightly integrated.

### kubectl metrics

```
sudo microk8s kubectl top nodes
```
```
NAME        CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s         159m         3%     2026Mi          53%
```

```
sudo microk8s kubectl top pods -n cppserver
```
```
NAME                                    CPU(cores)   MEMORY(bytes)
cppserver-5f66f5d57f-5mqz5              1m           3Mi
```

## Resources - highly recommended tutorials

* [MicroK8s in high-availability mode](https://microk8s.io/docs/clustering)
* [Kubernetes explained in 6 minutes](https://youtu.be/TlHvYWVUZyc)
* [Get started with Kubernetes](https://www.suse.com/c/rancher_blog/91081/)
* [75 Kubernetes concepts in short](https://medium.com/@bubu.tripathy/top-75-kubernetes-questions-and-answers-d677a0b87d79)

