# Configuring EFK to run with PersistentVolumes provisioned by the LocalStorage Operator on FYRE

NOTE: These instructions are a mix of the [LocalStorage Operator Guide](https://docs.openshift.com/container-platform/4.2/storage/persistent_storage/persistent-storage-local.html) and the [Deploying Cluster Logging Guide](https://docs.openshift.com/container-platform/4.3/logging/cluster-logging-deploying.html), but with more specific YAML examples to FYRE. Some parts of those 2 guides are also unneeded; these instructions will specific when to use those guides and when to use the instructions here.

It seems like ElasticSearch is not supported for NFS or distributed storage, and that the recommendation should be to use local disk storage, so these are instructions to get the EFK stack to run on PersistentVolumes (created for a specified local disk) provisioned by the LocalStorage Operator.

## Provision the Local Volumes Using the Local Storage Operator

PREREQUISITE: Have the LocalStorage Operator installed. Follow these [instructions](https://docs.openshift.com/container-platform/4.2/storage/persistent_storage/persistent-storage-local.html#local-storage-install_persistent-storage-local) to install the LocalStorage Operator. Return back to these instructions at the "Provision the local volumes" section because the YAML here is more specific, and you won't need the rest of the other guide.

After installing the LocalStorage Operator, create and modify the following YAML file to create your LocalVolume CR. This [section in the LocalStorage Operator guide](https://docs.openshift.com/container-platform/4.2/storage/persistent_storage/persistent-storage-local.html#local-volume-cr_persistent-storage-local) has information on what each of the definitions in the YAML are for.

```YAML
apiVersion: "local.storage.openshift.io/v1"
kind: "LocalVolume"
metadata:
  name: "local-disks"
  namespace: "local-storage" 
spec:
  nodeSelector: 
    nodeSelectorTerms:
    - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values: # replace these next 3 lines with your VMs worker nodes
          - worker0.ocpvm.os.fyre.ibm.com
          - worker1.ocpvm.os.fyre.ibm.com
          - worker2.ocpvm.os.fyre.ibm.com
  storageClassDevices:
    - storageClassName: "local-sc" # name of the storage class that will be created
      volumeMode: Filesystem 
      fsType: xfs # for elasticsearch, the recommendations should be xfs or ext4,this has only been tested with xfs   
      devicePaths: 
        - /path/to/device # in FYRE OCP 4.3, this disk /dev/vdb can be used, should have 500Gi of storage 

```

Create the `LocalVolume` resource in your OCP cluster with the YAML file you just made. 

```$ oc create -f <local-volume>.yaml ```

The creation of the resource should automatically create a `StorageClass` called `local-sc` as specified in the YAML. 
    
Six pods should be created: 3 diskmaker pods and 3 provisioner pods. You can see the pods in the OCP web console or with:

```$ oc get all -n local-storage```

```
NAME                                          READY   STATUS  RESTARTS   AGE
pod/local-disks-local-provisioner-h97hj       1/1     Running   0          46m
pod/local-disks-local-provisioner-j4mnn       1/1     Running   0          46m
pod/local-disks-local-provisioner-kbdnx       1/1     Running   0          46m
pod/local-disks-local-diskmaker-ldldw         1/1     Running   0          46m
pod/local-disks-local-diskmaker-lvrv4         1/1     Running   0          46m
pod/local-disks-local-diskmaker-phxdq         1/1     Running   0          46m
pod/local-storage-operator-54564d9988-vxvhx   1/1     Running   0          47m

NAME                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
service/local-storage-operator    ClusterIP   172.30.49.90     <none>        60000/TCP   47m

NAME                                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/local-disks-local-provisioner   3         3         3       3            3           <none>          46m
daemonset.apps/local-disks-local-diskmaker     3         3         3       3            3           <none>          46m

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/local-storage-operator   1/1     1            1           47m

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/local-storage-operator-54564d9988   1         1         1       47m
```


If the pods are working correctly, the creation of the `LocalVolume` CR should automatically provision three `PersistentVolumes` as well. You can see that those are running properly with:
         
```$ oc get pv```

```    
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
local-pv-1cec77cf   100Gi      RWO            Delete           Available           local-sc                88m
local-pv-2ef7cd2a   100Gi      RWO            Delete           Available           local-sc                82m
local-pv-3fa1c73    100Gi      RWO            Delete           Available           local-sc                48m
```

If you go into your web console to see Storage > Persistent Volumes, you will see that all three of the `PersistentVolumes` are availalable and unbound. When we create the ClusterLogging instance next, 3 Elasticsearch `PersistentVolumeClaims` will be created that will bind to the 3 PVs.

## Configuring the EFK Instance to Use the PersistentVolumes Provisioned by the Local Storage Operator

PREREQUISITE: Have the Elasticsearch Operator and the ClusterLogging Operator installed. Follow these [instructions](https://docs.openshift.com/container-platform/4.3/logging/cluster-logging-deploying.html) to install the Operators. Return to this guide when it asks you to create a ClusterLogging instance because this ClusterLogging instance is more specific.

After installing the Elasticsearch and ClusterLogging Operators, Create a ClusterLogging instance in the web console with the following YAML:

```YAML
apiVersion: "logging.openshift.io/v1"
kind: "ClusterLogging"
metadata:
  name: "instance" 
  namespace: "openshift-logging"
spec:
  managementState: "Managed"  
  logStore:
    type: "elasticsearch"  
    elasticsearch:
      nodeCount: 3 
      resources:
        limits:
          memory: 2Gi
        requests:
          cpu: 200m
          memory: 2Gi
      storage:
        storageClassName: local-sc #this is the storage class created by our LocalVolume resource
        size: 200G
      redundancyPolicy: "SingleRedundancy"
  visualization:
    type: "kibana"  
    kibana:
      replicas: 1
  curation:
    type: "curator"  
    curator:
      schedule: "30 3 * * *"
  collection:
    logs:
      type: "fluentd"  
      fluentd: {}
```
All the pods should come up and the 3 Elasticsearch PersistentVolumeClaims created should automatically search for and bind to the 3 PVs provisioned by the LocalStorage Operator pods.



