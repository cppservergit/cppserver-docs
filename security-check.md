# Automatic Kubernetes-native security check using Trivy and MicroK8s

In this tutorial you will use the convenient Trivy Kubernetes Operator created by the security experts Aqua Security Software Ltd, to execute an automatic scan of your cluster (the one created with the QuickStart tutorial) searching for vulnerabilities and producing detailed reports, all of these tasks will be executed via kubectl, like any other administrative task. It's assumed that you have under your control a VM named k8s with the single-node MicroK8s cluster running, as explained in the CPPServer QuickStart tutorial.

### Resources:

* [Trivy Operator website](https://aquasecurity.github.io/trivy-operator/v0.13.0/)
* [CPPServer QuickStart with MicroK8s](https://github.com/cppservergit/cppserver-docs/blob/main/quickstart.md)

## Step 1: Install Trivy

Enter in your cluster Linux terminal and execute:
```
sudo microk8s kubectl apply -f https://raw.githubusercontent.com/aquasecurity/trivy-operator/v0.13.0/deploy/static/trivy-operator.yaml
```

Execute this to test if the Operator is running:
```
sudo microk8s kubectl get deployment -n trivy-system
```

You should see a response like this, if not, wait a few seconds and repeat until READY is 1/1.
```
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
trivy-operator   1/1     1            1           23h
```

## Step 2: Generate configuration audit report

This tool verifies Kubernetes-specific security checks.

Execute:
```
sudo microk8s kubectl get configauditreports -o wide -n cppserver
```

Output:
```
NAME                              SCANNER   AGE   CRITICAL   HIGH   MEDIUM   LOW
service-cppserver                 Trivy     23h   0          0      0        0
cronjob-cppjob                    Trivy     23h   0          0      0        0
replicaset-cppserver-674b6775cd   Trivy     22h   0          0      1        0
```

## Step 3: Print report detail

Good results, only one MEDIUM alert on the object replicaset-cppserver-674b6775cd, please note that in your environment the name of this object will vary. 
We can describe the object to obtain the detail of every alert:
```
sudo microk8s kubectl describe configauditreport -n cppserver replicaset-cppserver-674b6775cd
```

Output:
```
Name:         replicaset-cppserver-674b6775cd
Namespace:    cppserver
Labels:       plugin-config-hash=659b7b9c46
              resource-spec-hash=85bb549d5d
              trivy-operator.resource.kind=ReplicaSet
              trivy-operator.resource.name=cppserver-674b6775cd
              trivy-operator.resource.namespace=cppserver
Annotations:  trivy-operator.aquasecurity.github.io/report-ttl: 24h0m0s
API Version:  aquasecurity.github.io/v1alpha1
Kind:         ConfigAuditReport
Metadata:
  Creation Timestamp:  2023-04-07T01:27:51Z
  Generation:          1
  Managed Fields:
    API Version:  aquasecurity.github.io/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:trivy-operator.aquasecurity.github.io/report-ttl:
        f:labels:
          .:
          f:plugin-config-hash:
          f:resource-spec-hash:
          f:trivy-operator.resource.kind:
          f:trivy-operator.resource.name:
          f:trivy-operator.resource.namespace:
        f:ownerReferences:
          .:
          k:{"uid":"51da28a2-3334-4723-9294-76718636d944"}:
      f:report:
        .:
        f:checks:
        f:scanner:
          .:
          f:name:
          f:vendor:
          f:version:
        f:summary:
          .:
          f:criticalCount:
          f:highCount:
          f:lowCount:
          f:mediumCount:
        f:updateTimestamp:
    Manager:    trivy-operator
    Operation:  Update
    Time:       2023-04-07T01:27:51Z
  Owner References:
    API Version:           apps/v1
    Block Owner Deletion:  false
    Controller:            true
    Kind:                  ReplicaSet
    Name:                  cppserver-674b6775cd
    UID:                   51da28a2-3334-4723-9294-76718636d944
  Resource Version:        20286
  UID:                     3e47520a-3008-4448-996a-150e63d30d47
Report:
  Checks:
    Category:     Kubernetes Security Check
    Check ID:     KSV023
    Description:  HostPath volumes must be forbidden.
    Messages:
      ReplicaSet 'cppserver-674b6775cd' should not set 'spec.template.volumes.hostPath'
    Severity:  MEDIUM
    Success:   false
    Title:     hostPath volumes mounted
  Scanner:
    Name:     Trivy
    Vendor:   Aqua Security
    Version:  0.13.0
  Summary:
    Critical Count:  0
    High Count:      0
    Low Count:       0
    Medium Count:    1
  Update Timestamp:  2023-04-07T01:27:51Z
Events:              <none>
```

This is the relation between the report and the detail:
![Security Check](https://cppserver.com/trivy-report.png)

In this case this is false-positive because the deployment of CPPServer for the QuickStart is running on a single-node cluster for testing and development purposes and for that reason it does use hostPath volumes (local filesystem) to store the website and the blobs, on production environments the website could reside inside the container image and the blob storage could be provided by a cluster-wide solution, like a NFS server, a Kubernetes-native storage, or a Cloud-provider storage. As you can see from the good audit results above, CPPServer deployment file (cppserver.yaml) does already include recommended security practices for Kubernetes.

If you want to see the contents of cppserver.yaml, it's stored in /home/ubuntu on the VM used for the QuickStart tutorial, thanks to Trivy and its detailed reports, we were able to add all the recommended security-related fixes to the YAML file. To view your current cppserver.yaml:
```
cat $HOME/cppserver.yaml
```

We can do more about security, like adding a Network Policy to block direct access to the CPPServer service and Pods, so that only the Ingress can access it. We don't have NetworkPolicy in the QuickStart deployment because MicroK8s won't provide by default this feature on a single-node cluster, to test NetworkPolicy with MicroK8s we need a high-availability (HA) cluster, with at least 3 nodes, it's easy to build one using LXD containers on a single Multipass VM to host the whole cluster, we cover this in another article on this repository.
