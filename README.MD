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

## unzip / untar
```
[root@orfdns ~]# cd envoy_contour/
[root@orfdns envoy_contour]# gzip -d tkg-extensions-manifests-v1.3.1-vmware.1.tar.gz
[root@orfdns envoy_contour]# tar xvf tkg-extensions-manifests-v1.3.1-vmware.1.tar
[root@orfdns envoy_contour]# cd tkg-extensions-v1.3.1+vmware.1
[root@orfdns tkg-extensions-v1.3.1+vmware.1]# 
```

## log onto the new cluster
```
[root@orfdns tkg-extensions-v1.3.1+vmware.1]# alias l2540
alias l2540='kubectl vsphere login --server 192.168.5.40 \
                --vsphere-username administrator@vsphere.local \
                --managed-cluster-namespace namespace1000 \
                --managed-cluster-name tkg-berlin \
                --insecure-skip-tls-verify'
[root@orfdns tkg-extensions-v1.3.1+vmware.1]# alias l2540paris
alias l2540paris='kubectl vsphere login --server 192.168.5.40 \
                --vsphere-username administrator@vsphere.local \
                --managed-cluster-namespace namespace1000 \
                --managed-cluster-name tkg-paris \
                --insecure-skip-tls-verify'
[root@orfdns tkg-extensions-v1.3.1+vmware.1]# l2540paris

Password: 
Logged in successfully.

You have access to the following contexts:
   192.168.5.40
   namespace1000
   tkg-berlin
   tkg-paris

If the context you wish to use is not in this list, you may need to try
logging in again later, or contact your cluster administrator.

To change context, use `kubectl config use-context <workload name>`
[root@orfdns tkg-extensions-v1.3.1+vmware.1]# kubectl config use-context tkg-paris
Switched to context "tkg-paris".
[root@orfdns tkg-extensions-v1.3.1+vmware.1]# k get nodes
NAME                                       STATUS   ROLES                  AGE   VERSION
tkg-paris-control-plane-2hv5b              Ready    control-plane,master   15m   v1.20.2+vmware.1
tkg-paris-workers-5z2vw-5ccdc7d4cb-594n5   Ready    <none>                 11m   v1.20.2+vmware.1
tkg-paris-workers-5z2vw-5ccdc7d4cb-qzv8j   Ready    <none>                 11m   v1.20.2+vmware.1
```
## Permissions
```
[root@orfdns ~]# k apply -f ./authorize-psp-for-gc-service-accounts.yaml  
clusterrole.rbac.authorization.k8s.io/psp:privileged created
clusterrolebinding.rbac.authorization.k8s.io/all:psp:privileged created
[root@orfdns ~]# 
[root@orfdns ~]# 
[root@orfdns ~]# 
[root@orfdns ~]# cat authorize-psp-for-gc-service-accounts.yaml  
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: psp:privileged
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - vmware-system-privileged
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: all:psp:privileged
roleRef:
  kind: ClusterRole
  name: psp:privileged
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```

## Deploy
```
[root@orfdns tkg-extensions-v1.3.1+vmware.1]# cd ..
[root@orfdns envoy_contour]# cd tkg-extensions-v1.3.1+vmware.1/extensions
[root@orfdns extensions]# kubectl apply -f kapp-controller.yaml
namespace/tkg-system created
serviceaccount/kapp-controller-sa created
customresourcedefinition.apiextensions.k8s.io/apps.kappctrl.k14s.io created
deployment.apps/kapp-controller created
clusterrole.rbac.authorization.k8s.io/kapp-controller-cluster-role created
clusterrolebinding.rbac.authorization.k8s.io/kapp-controller-cluster-role-binding created
```
```
[root@orfdns extensions]# cd  ..
[root@orfdns tkg-extensions-v1.3.1+vmware.1]# kubectl apply -f cert-manager/
namespace/cert-manager created
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created
serviceaccount/cert-manager-cainjector created
serviceaccount/cert-manager created
serviceaccount/cert-manager-webhook created
clusterrole.rbac.authorization.k8s.io/cert-manager-cainjector created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-issuers created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-clusterissuers created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-certificates created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-orders created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-challenges created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-ingress-shim created
clusterrole.rbac.authorization.k8s.io/cert-manager-view created
clusterrole.rbac.authorization.k8s.io/cert-manager-edit created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-cainjector created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-issuers created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-clusterissuers created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-certificates created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-orders created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-challenges created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-ingress-shim created
role.rbac.authorization.k8s.io/cert-manager-cainjector:leaderelection created
role.rbac.authorization.k8s.io/cert-manager:leaderelection created
role.rbac.authorization.k8s.io/cert-manager-webhook:dynamic-serving created
rolebinding.rbac.authorization.k8s.io/cert-manager-cainjector:leaderelection created
rolebinding.rbac.authorization.k8s.io/cert-manager:leaderelection created
rolebinding.rbac.authorization.k8s.io/cert-manager-webhook:dynamic-serving created
service/cert-manager created
service/cert-manager-webhook created
deployment.apps/cert-manager-cainjector created
deployment.apps/cert-manager created
deployment.apps/cert-manager-webhook created
mutatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created
validatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created
```









## Reference documentation
```
Doc: https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-FA305850-ED37-48C6-99E9-5B6996376F0E.html
Install TKG extensions pre-requs
https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-36C7D7CB-312F-49B6-B542-1D0DBC550198.html

```
