# Automatic Kubernetes-native security check using Trivy and MicroK8s

In this tutorial you will use the convenient Trivy Kubernetes Operator created by the security experts Aqua Security Software Ltd, to execute an automatic scan of your cluster (the one created with the QuickStart tutorial) searching for vulnerabilities and producing detailed reports, all of these tasks will be executed via kubectl, like any other administrative task.

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

You should see a response like this, if not, repeat until READY is 1/1.
```
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
trivy-operator   1/1     1            1           23h
```

## Step 2: Generate vulnerability report

Execute:
```
sudo microk8s kubectl get vulnerabilityreports -o wide -n cppserver
```

This will scan the CPPServer docker image, for this case it will detect some (15) LOW vulnerabilities, from the Ubuntu 22.04 base image, these are not relevant, not only because of the low risk classification, but also because these are located in software modules that mostly won't be used during the container execution.
```
NAME                                        REPOSITORY            TAG    SCANNER   AGE   CRITICAL   HIGH   MEDIUM   LOW   UNKNOWN
replicaset-cppserver-674b6775cd-cppserver   cppserver/pgsql       1.01   Trivy     21h   0          0      0        15    0
cronjob-cppjob-cppjob                       cppserver/pgsql-job   1.00   Trivy     23h   0          0      0        15    0
```

Each of the objects listed under the NAME header is a report, in the next section you can print its detail, you can do it for each report listed. 

### Print detailed report

Let's see the vilnerabilities list in the pgsql-job container image:
```
sudo microk8s kubectl describe vulnerabilityreport cronjob-cppjob-cppjob -n cppserver
```

Output:
```
Name:         cronjob-cppjob-cppjob
Namespace:    cppserver
Labels:       resource-spec-hash=84bdff4f4f
              trivy-operator.container.name=cppjob
              trivy-operator.resource.kind=CronJob
              trivy-operator.resource.name=cppjob
              trivy-operator.resource.namespace=cppserver
Annotations:  trivy-operator.aquasecurity.github.io/report-ttl: 24h0m0s
API Version:  aquasecurity.github.io/v1alpha1
Kind:         VulnerabilityReport
Metadata:
  Creation Timestamp:  2023-04-06T23:47:20Z
  Generation:          2
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
          f:resource-spec-hash:
          f:trivy-operator.container.name:
          f:trivy-operator.resource.kind:
          f:trivy-operator.resource.name:
          f:trivy-operator.resource.namespace:
        f:ownerReferences:
          .:
          k:{"uid":"1fffb97c-6c8a-4638-8aff-474055df1103"}:
      f:report:
        .:
        f:artifact:
          .:
          f:repository:
          f:tag:
        f:registry:
          .:
          f:server:
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
          f:noneCount:
          f:unknownCount:
        f:updateTimestamp:
        f:vulnerabilities:
    Manager:    trivy-operator
    Operation:  Update
    Time:       2023-04-07T01:29:18Z
  Owner References:
    API Version:           batch/v1
    Block Owner Deletion:  false
    Controller:            true
    Kind:                  CronJob
    Name:                  cppjob
    UID:                   1fffb97c-6c8a-4638-8aff-474055df1103
  Resource Version:        20570
  UID:                     9ee35960-9b78-4530-af67-979c06215d7c
Report:
  Artifact:
    Repository:  cppserver/pgsql-job
    Tag:         1.00
  Registry:
    Server:  index.docker.io
  Scanner:
    Name:     Trivy
    Vendor:   Aqua Security
    Version:  0.39.0
  Summary:
    Critical Count:  0
    High Count:      0
    Low Count:       15
    Medium Count:    0
    None Count:      0
    Unknown Count:   0
  Update Timestamp:  2023-04-07T01:29:18Z
  Vulnerabilities:
    Fixed Version:
    Installed Version:  5.1-6ubuntu1
    Links:
    Primary Link:       https://avd.aquasec.com/nvd/cve-2022-3715
    Resource:           bash
    Score:              7.8
    Severity:           LOW
    Target:
    Title:              bash: a heap-buffer-overflow in valid_parameter_transform
    Vulnerability ID:   CVE-2022-3715
    Fixed Version:
    Installed Version:  8.32-4.1ubuntu1
    Links:
    Primary Link:       https://avd.aquasec.com/nvd/cve-2016-2781
    Resource:           coreutils
    Score:              6.5
    Severity:           LOW
    Target:
    Title:              coreutils: Non-privileged session can escape to the parent session in chroot
    Vulnerability ID:   CVE-2016-2781
    Fixed Version:
    Installed Version:  2.2.27-3ubuntu2.1
    Links:
    Primary Link:       https://avd.aquasec.com/nvd/cve-2022-3219
    Resource:           gpgv
    Score:              5.5
    Severity:           LOW
    Target:
    Title:              gnupg: denial of service issue (resource consumption) using compressed packets
    Vulnerability ID:   CVE-2022-3219
    Fixed Version:
    Installed Version:  2.35-0ubuntu3.1
    Links:
    Primary Link:       https://avd.aquasec.com/nvd/cve-2016-20013
    Resource:           libc-bin
    Score:              7.5
    Severity:           LOW
    Target:
    Title:
    Vulnerability ID:   CVE-2016-20013
    Fixed Version:
    Installed Version:  2.35-0ubuntu3.1
    Links:
    Primary Link:       https://avd.aquasec.com/nvd/cve-2016-20013
    Resource:           libc6
    Score:              7.5
    Severity:           LOW
    Target:
    Title:
    Vulnerability ID:   CVE-2016-20013
    Fixed Version:
    Installed Version:  6.3-2
    Links:
    Primary Link:       https://avd.aquasec.com/nvd/cve-2022-29458
    Resource:           libncurses6
    Score:              7.1
    Severity:           LOW
    Target:
    Title:              ncurses: segfaulting OOB read
    Vulnerability ID:   CVE-2022-29458
    Fixed Version:
    Installed Version:  6.3-2
    Links:
    Primary Link:       https://avd.aquasec.com/nvd/cve-2022-29458
    Resource:           libncursesw6
    Score:              7.1
    Severity:           LOW
    Target:
    Title:              ncurses: segfaulting OOB read
    Vulnerability ID:   CVE-2022-29458
    Fixed Version:
    Installed Version:  2:8.39-13ubuntu0.22.04.1
    Links:
    Primary Link:       https://avd.aquasec.com/nvd/cve-2017-11164
    Resource:           libpcre3
    Score:              7.5
    Severity:           LOW
    Target:
    Title:              pcre: OP_KETRMAX feature in the match function in pcre_exec.c
    Vulnerability ID:   CVE-2017-11164
    Fixed Version:
    Installed Version:  3.0.2-0ubuntu1.8
    Links:
    Primary Link:       https://avd.aquasec.com/nvd/cve-2022-3996
    Resource:           libssl3
    Score:              7.5
    Severity:           LOW
    Target:
    Title:              openssl: double locking leads to denial of service
    Vulnerability ID:   CVE-2022-3996
    Fixed Version:
    Installed Version:  3.0.2-0ubuntu1.8
    Links:
    Primary Link:       https://avd.aquasec.com/nvd/cve-2023-0464
    Resource:           libssl3
    Score:              7.5
    Severity:           LOW
    Target:
    Title:              openssl: Denial of service by excessive resource usage in verifying X509 policy constraints
    Vulnerability ID:   CVE-2023-0464
    Fixed Version:
    Installed Version:  3.0.2-0ubuntu1.8
    Links:
    Primary Link:       https://avd.aquasec.com/nvd/cve-2023-0465
    Resource:           libssl3
    Score:              5.3
    Severity:           LOW
    Target:
    Title:              openssl: Invalid certificate policies in leaf certificates are silently ignored
    Vulnerability ID:   CVE-2023-0465
    Fixed Version:
    Installed Version:  3.0.2-0ubuntu1.8
    Links:
    Primary Link:       https://avd.aquasec.com/nvd/cve-2023-0466
    Resource:           libssl3
    Score:              5.3
    Severity:           LOW
    Target:
    Title:              openssl: Certificate policy check not enabled
    Vulnerability ID:   CVE-2023-0466
    Fixed Version:
    Installed Version:  6.3-2
    Links:
    Primary Link:       https://avd.aquasec.com/nvd/cve-2022-29458
    Resource:           libtinfo6
    Score:              7.1
    Severity:           LOW
    Target:
    Title:              ncurses: segfaulting OOB read
    Vulnerability ID:   CVE-2022-29458
    Fixed Version:
    Installed Version:  6.3-2
    Links:
    Primary Link:       https://avd.aquasec.com/nvd/cve-2022-29458
    Resource:           ncurses-base
    Score:              7.1
    Severity:           LOW
    Target:
    Title:              ncurses: segfaulting OOB read
    Vulnerability ID:   CVE-2022-29458
    Fixed Version:
    Installed Version:  6.3-2
    Links:
    Primary Link:      https://avd.aquasec.com/nvd/cve-2022-29458
    Resource:          ncurses-bin
    Score:             7.1
    Severity:          LOW
    Target:
    Title:             ncurses: segfaulting OOB read
    Vulnerability ID:  CVE-2022-29458
Events:                <none>
```

## Step 3: Generate configuration audit report

This is the most interesting report, because it does verify Kubernetes-specific security checks.

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

### Print report detail

Nice! only 1 MEDIUM alert, as in the previous case, we can describe the object to obtain the detail:
```
sudo microk8s kubectl describe configauditreport replicaset-cppserver-674b6775cd -n cppserver
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
![](https://cppserver.com/trivy-report.png)

In this case this is false-positive because the deployment of CPPServer for the QuickStart is running on a single-node cluster for testing and development tasks and for that reason it does use hostPath volumes (local filesystem) to store the website and the blobs, on production environments the website could reside inside the container image and the blob storage could be provided by a cluster-wide solution, like traditional NFS server, a Kubernetes-native storage, or a Cloud-provider storage. As you can see from the good audit results above, CPPServer deployment file (cppserver.yaml) does already include recommended security practices for Kubernetes.




