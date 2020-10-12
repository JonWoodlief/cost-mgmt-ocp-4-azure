# Openshift 4.5 cost management on Azure

1. install/configure metering operator- https://docs.openshift.com/container-platform/4.5/metering/metering-about-metering.html
2. Install/configure cost-management operator- https://access.redhat.com/documentation/en-us/openshift_container_platform/4.5/pdf/getting_started_with_cost_management/OpenShift_Container_Platform-4.5-Getting_started_with_cost_management-en-US.pdf
3. Hook up Azure cost exports to Red Hat cost management
   

## Setup File storageclass for ReadWriteMany volumes

You can skip this step if you already have file storage available

Create clusterrole needed for Azure storage provisioner-

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: persistent-volume-binder
rules:
- apiGroups: ['']
  resources: ['secrets']
  verbs:     ['get','create']
```

Add role to storageclass's serviceaccount

```bash
oc adm policy add-cluster-role-to-user persistent-volume-binder system:serviceaccount:kube-system:persistent-volume-binder
```

Create storage account

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: azure-file-metering
mountOptions:
  - uid=1000620000
  - gid=1000620000
provisioner: kubernetes.io/azure-file
parameters:
  location: eastus
  skuName: Standard_LRS
reclaimPolicy: Delete
volumeBindingMode: Immediate

```

You'll see that we've included a uid and gid for this storageclass. This isn't guaranteed to work on your deployment- of metering operator. After installing- we will check for errors- and if there are disk permissions issues, we can change this storageclass to make sure the uid matches up with the uid which our hive storage pods are using.
## Install metering operator

create openshift-metering namespace-

```yaml
kind: Namespace
metadata:
  name: openshift-metering
  annotations:
    openshift.io/node-selector: ""
  labels:
    openshift.io/cluster-monitoring: "true"
```


Deploy openshift-metering through operator hub into openshift-metering project. Default deployment options work fine

```yaml
apiVersion: metering.openshift.io/v1
kind: MeteringConfig
metadata:
  name: "operator-metering"
  namespace: "openshift-metering"
spec:
  storage:
    type: "hive"
    hive:
      type: "sharedPVC"
      sharedPVC:
        createPVC: true
        storageClass: azure-file-metering
        size: 5Gi
```

This should create a few pods. This stack can take a lot of resources- make sure the cluster has room to shedule them all

```
cpat@privatevnetbastion:~$ oc get pods
NAME                                  READY   STATUS    RESTARTS   AGE
hive-metastore-0                      2/2     Running   0          106s
hive-server-0                         3/3     Running   0          107s
metering-operator-5b7c88f6fd-gm24r    2/2     Running   0          3m59s
presto-coordinator-0                  2/2     Running   0          85s
reporting-operator-5f8b9dc6db-7fvbt   1/2     Running   0          60s
```

To verify that openshift-metering is working- run an 'oc get datasources' and check to make sure that some fields like EARLIEST METRIC are populating. Not all will populate quickly- but if after a few minutes, all fields are still totally empty, something may be wrong. Most likely a uid mismatch between the storage and the pods mounting the storage. If using Azure file storage- jump to the bottom of this document. if not- https://docs.openshift.com/container-platform/4.5/metering/metering-troubleshooting-debugging.html

## Install Cost Management Operator

On the openshift web console, navigate to operatorhub

search and install the cost management operator. Follow the instructions on operatorhub and https://access.redhat.com/documentation/en-us/openshift_container_platform/4.5/pdf/getting_started_with_cost_management/OpenShift_Container_Platform-4.5-Getting_started_with_cost_management-en-US.pdf

At a high level- you'll create an auth secret to allow you to upload data to your redhat account, and then you'll create costmanagement custom resources


If you use the token as your authentication- you only need to set the token in your auth-secret. Make sure to delete the line that says 'authentication: basic' in your cost management resource yaml if you're using the token to authenticate.



## Hook up Azure source to cost management

https://access.redhat.com/documentation/en-us/openshift_container_platform/4.5/pdf/getting_started_with_cost_management/OpenShift_Container_Platform-4.5-Getting_started_with_cost_management-en-US.pdf
Follow documentation, very straightforward



## Troubleshooting- Empty datasource table- Azure File

Get the reporting operator logs with the following command-

```bash
oc -n openshift-metering logs -f "$(oc -n openshift-metering get pods -l app=reporting-operator -o name | cut -c 5-)" -c reporting-operator
```

If you see something like the following in the output-

```bash
time="2020-10-11T17:08:55Z" level=info msg="error syncing ReportDataSource \"openshift-metering/pod-request-memory-bytes\", dropping out of the queue" ReportDataSource=openshift-metering/pod-request-memory-bytes app=metering component=reportDataSourceWorker error="ImportFromLastTimestamp errored: failed to store Prometheus metrics into table hive.metering.datasource_openshift_metering_pod_request_memory_bytes for the range 2020-10-11 15:08:00 +0000 UTC to 2020-10-11 15:13:00 +0000 UTC: failed to store metrics into presto: presto SQL error: presto: query failed (200 OK): \"io.prestosql.spi.PrestoException: Failed to rename file:/tmp/presto-reporting-operator/17094141-7980-4678-b604-e6b2f179758e/dt=2020-10-11 to file:/user/hive/warehouse/metering.db/datasource_openshift_metering_pod_request_memory_bytes/dt=2020-10-11\"" logID=yNpUMpayhk

```

There is most likely a UID mismatch with your azure file storage volume and the container running it.

One solution to this is to provision a new azure storage class to provision a storage volume with the correct UID.

To get the UID run the following command-

```bash
oc get pod hive-server-0 -o yaml | grep runAsUser
```

In my case it was 1000620000. I can then provision a new azure storageclass to supply a PV with this UID with the following-

```YAML
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: azure-file3
mountOptions:
  - uid=1000620000
  - gid=1000620000
provisioner: kubernetes.io/azure-file
parameters:
  location: eastus
  skuName: Standard_LRS
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

Delete the meteringconfig using '`oc delete meteringconfig operator-metering` and then edit your meteringconfig yaml to use the new storageclass. Apply the new yaml and you should be good to go