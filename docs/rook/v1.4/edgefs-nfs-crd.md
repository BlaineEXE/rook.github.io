---
title: Scale-Out NFS CRD
weight: 4300
indent: true
---

# EdgeFS Scale-Out NFS CRD

[Deprecated](https://github.com/rook/rook/issues/5823#issuecomment-703834989)

Rook allows creation and customization of EdgeFS NFS filesystems through the custom resource definitions (CRDs).
The following settings are available for customization of EdgeFS NFS services.

## Sample

```yaml
apiVersion: edgefs.rook.io/v1
kind: NFS
metadata:
  name: nfs01
  namespace: rook-edgefs
spec:
  instances: 3
  #relaxedDirUpdates: true
  #chunkCacheSize: 1Gi
  # A key/value list of annotations
  annotations:
  #  key: value
  placement:
  #  nodeAffinity:
  #    requiredDuringSchedulingIgnoredDuringExecution:
  #      nodeSelectorTerms:
  #      - matchExpressions:
  #        - key: role
  #          operator: In
  #          values:
  #          - nfs-node
  #  tolerations:
  #  - key: nfs-node
  #    operator: Exists
  #  podAffinity:
  #  podAntiAffinity:
  #resourceProfile: embedded
  resources:
  #  limits:
  #    cpu: "500m"
  #    memory: "1024Mi"
  #  requests:
  #    cpu: "500m"
  #    memory: "1024Mi"
```

## Metadata

* `name`: The name of the NFS system to create, which must match existing EdgeFS service.
* `namespace`: The namespace of the Rook cluster where the NFS service is created.
* `instances`: The number of active NFS service instances. EdgeFS NFS service is Multi-Head capable, such so that multiple PODs can mount same tenant's buckets via different endpoints. [EdgeFS CSI provisioner](edgefs-csi.md) orchestrates distribution and load balancing across NFS service instances in round-robin or random policy ways.
* `relaxedDirUpdates`: If set to `true` then it will significantly improve performance of directory operations by defering updates, guaranteeing eventual directory consistency. This option is recommended when a bucket exported via single NFS instance and it is not a destination for ISGW Link synchronization.
* `chunkCacheSize`: Limit amount of memory allocated for dynamic chunk cache. By default NFS pod uses up to 75% of available memory as chunk caching area. This option can influence this allocation strategy.
* `annotations`: Key value pair list of annotations to add.
* `placement`: The NFS PODs can be given standard Kubernetes placement restrictions with `nodeAffinity`, `tolerations`, `podAffinity`, and `podAntiAffinity` similar to placement defined for daemons configured by the [cluster CRD](/cluster/examples/kubernetes/edgefs/cluster.yaml).
* `resourceProfile`: NFS pod resource utilization profile (Memory and CPU). Can be `embedded` or `performance` (default). In case of `performance` an NFS pod trying to increase amount of internal I/O resources that results in higher performance at the cost of additional memory allocation and more CPU load. In `embedded` profile case, NFS pod gives preference to preserving memory over I/O and limiting chunk cache (see `chunkCacheSize` option). The `performance` profile is the default unless cluster wide `embedded` option is defined.
* `resources`: Set resource requests/limits for the NFS Pod(s), see [Resource Requirements/Limits](edgefs-cluster-crd.md#resource-requirementslimits).

## Setting up EdgeFS namespace and tenant

For more detailed instructions please refer to [EdgeFS Wiki](https://github.com/Nexenta/edgefs/wiki).

Below is an exampmle procedure to get things initialized and configured.

Before new local namespace (or local site) can be used, it has to be initialized with FlexHash and special purpose root object.

FlexHash consists of dynamically discovered configuration and checkpoint of accepted distribution table. FlexHash is responsible for I/O direction and plays important role in dynamic load balancing logic. It defines so-called Negotiating Groups (typically across zoned 8-24 disks) and final table distribution across all the participating components, e.g. data nodes, service gateways and tools.

Root object holds system information and table of namespaces registered to a local site. Root object is always local and never shared between the sites.

To initialize system and prepare logical definitions, login to the toolbox as shown in this example:

```console
kubectl get po --all-namespaces | grep edgefs-mgr
kubectl exec -it -n rook-edgefs rook-edgefs-mgr-6cb9598469-czr7p -- env COLUMNS=$COLUMNS LINES=$LINES TERM=linux toolbox
```

Assumption at this point is that nodes are all configured and can be seen via the following command:

```console
efscli system status
```

1. Initialize EdgeFS cluster:

Verify that HW (or better say emulated in this case) configuration look normal and accept it:

```console
efscli system init
```

At this point new dynamically discovered configuration checkpoint will be created at $NEDGE_HOME/var/run/flexhash-checkpoint.json
This will also create system "root" object, holding Site's Namespace. Namespace may consist of more then single region.

2. Create new local namespace (or we also call it "Region" or "Segment"):

```console
efscli cluster create Hawaii
```

3. Create logical tenants of cluster namespace "Hawaii", also buckets if needed:

```console
efscli tenant create Hawaii/Cola
efscli bucket create Hawaii/Cola/bk1
efscli tenant create Hawaii/Pepsi
efscli bucket create Hawaii/Pepsi/bk1
```

Now cluster is setup, services can be now created and attached to CSI provisioner.

4. Create NFS service objects for tenants:

```console
efscli service create nfs nfs-cola
efscli service serve nfs-cola Hawaii/Cola/bk1
efscli service create nfs nfs-pepsi
efscli service serve nfs-pepsi Hawaii/Pepsi/bk1
```

5. Create your EdgeFS NFS objects:

```yaml
apiVersion: edgefs.rook.io/v1
kind: NFS
metadata:
  name: nfs-cola
  namespace: rook-edgefs
spec:
  instances: 1
```

```yaml
apiVersion: edgefs.rook.io/v1
kind: NFS
metadata:
  name: nfs-pepsi
  namespace: rook-edgefs
spec:
  instances: 1
```

At this point two NFS services should be available. Verify that showmount command can see service (substitute CLUSTERIP with corresponding entry from `kubectl get svc` command):

```console
kubectl get svc --all-namespaces
showmount -e CLUSTERIP
```
