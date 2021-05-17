# Using Contour and Envoy and a DNS Star record to point to one IP but have many services

## Create a custer
```
[root@orfdns 7u2a]# k apply -f ./clusterparis.yaml 
tanzukubernetescluster.run.tanzu.vmware.com/tkg-paris created
```

## Here is the yaml file
```
# clusterparis.yaml
---
apiVersion: run.tanzu.vmware.com/v1alpha1
kind: TanzuKubernetesCluster
metadata:
  name: tkg-paris
  namespace: namespace1000
spec:
  distribution:
    version: v1.20.2
  topology:
    controlPlane:
      count: 1
      class: best-effort-medium
      storageClass: pacific-gold-storage-policy
      volumes:
        - name: etcd
          mountPath: /var/lib/etcd
          capacity:
            storage: 4Gi
    workers:
      count: 2
      class: best-effort-medium
      storageClass: pacific-gold-storage-policy
      volumes:
        - name: containerd
          mountPath: /var/lib/containerd
          capacity:
            storage: 30Gi
  settings:
    network:
      services:
        cidrBlocks: ["198.51.100.0/24"]
      pods:
        cidrBlocks: ["192.0.2.0/22"]
    storage:
      classes: ["pacific-gold-storage-policy"]
      defaultClass: pacific-gold-storage-policy
 ```
 ## Check on cluster
 ```
 [root@orfdns 7u2a]# k get tkc
NAME         CONTROL PLANE   WORKER   DISTRIBUTION                      AGE     PHASE      TKR COMPATIBLE   UPDATES AVAILABLE
tkg-berlin   1               2        v1.18.15+vmware.1-tkg.1.600e412   3d18h   running    True             [1.19.7+vmware.1-tkg.2.f52f85a 1.18.15+vmware.1-tkg.2.ebf6117]
tkg-paris    1               2        v1.20.2+vmware.1-tkg.2.3e10706    3m56s   creating   True             
[root@orfdns 7u2a]# 
```
 
## Create directory and copy downloaded extentions
```
Download:  https://www.vmware.com/go/get-tkg
		tkg-extensions-manifests-v1.3.1-vmware.1.tar.gz
```

```
[root@orfdns ~]# mkdir envoy_contour
[root@orfdns ~]# cp /tmp/tkg-extensions-manifests-v1.3.1-vmware.1.tar.gz envoy_contour/.
[root@orfdns ~]#
```










## Reference documentation
```
Doc: https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-FA305850-ED37-48C6-99E9-5B6996376F0E.html
```
