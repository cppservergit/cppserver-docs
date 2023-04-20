# Common administrative tasks

These are step-by-step instructions to execute the most common Admin tasks, but also those tasks that will be frequently required during development, like adding a microservice definition and then restart the Pods of CPPServer.

This tasks assume that you have a working cluster according to the QuickStart tutorial available in this Repo.

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

In the next section we will check the logs of the CPPServer Pod to verify the new value.

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






