# Using Contour and Envoy and a DNS Star record to point to one IP but have many services

Create a custer
```
[root@orfdns 7u2a]# k apply -f ./clusterparis.yaml 
tanzukubernetescluster.run.tanzu.vmware.com/tkg-paris created
```

Here is the yaml file
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
 
 
