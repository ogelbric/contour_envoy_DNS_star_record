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

## namespace
```
[root@orfdns tkg-extensions-v1.3.1+vmware.1]# cd extensions/ingress/contour
[root@orfdns contour]# kubectl apply -f namespace-role.yaml
namespace/tanzu-system-ingress created
serviceaccount/contour-extension-sa created
role.rbac.authorization.k8s.io/contour-extension-role created
rolebinding.rbac.authorization.k8s.io/contour-extension-rolebinding created
clusterrole.rbac.authorization.k8s.io/contour-extension-cluster-role created
clusterrolebinding.rbac.authorization.k8s.io/contour-extension-cluster-rolebinding created
[root@orfdns contour]# 
[root@orfdns contour]# k get ns
NAME                           STATUS   AGE
cert-manager                   Active   11m
default                        Active   37m
kube-node-lease                Active   37m
kube-public                    Active   37m
kube-system                    Active   37m
tanzu-system-ingress           Active   78s
tkg-system                     Active   12m
vmware-system-auth             Active   37m
vmware-system-cloud-provider   Active   37m
vmware-system-csi              Active   37m
```
## Contour
```
[root@orfdns contour]# cp vsphere/contour-data-values-lb.yaml.example vsphere/contour-data-values.yaml
[root@orfdns contour]# kubectl create secret generic contour-data-values --from-file=values.yaml=vsphere/contour-data-values.yaml -n tanzu-system-ingress
secret/contour-data-values created
[root@orfdns contour]# k get secrets -A | grep ingress
tanzu-system-ingress           contour-data-values                              Opaque                                1      2s
tanzu-system-ingress           contour-extension-sa-token-jfqhp                 kubernetes.io/service-account-token   3      2m58s
tanzu-system-ingress           default-token-v9gfd                              kubernetes.io/service-account-token   3      2m58s
[root@orfdns contour]# kubectl apply -f contour-extension.yaml
app.kappctrl.k14s.io/contour created
[root@orfdns contour]# k get pods -n tanzu-system-ingress
NAME                      READY   STATUS              RESTARTS   AGE
contour-85b79dfff-jbj8k   0/1     ContainerCreating   0          8s
contour-85b79dfff-vqrrr   0/1     ContainerCreating   0          7s
envoy-d44kw               0/2     Init:0/1            0          8s
envoy-lppjw               0/2     Init:0/1            0          8s

....time...

[root@orfdns contour]# k get pods -n tanzu-system-ingress
NAME                      READY   STATUS    RESTARTS   AGE
contour-85b79dfff-jbj8k   1/1     Running   0          97s
contour-85b79dfff-vqrrr   1/1     Running   0          96s
envoy-d44kw               2/2     Running   0          97s
envoy-lppjw               2/2     Running   0          97s
```

## Check the ingress
```
[root@orfdns contour]# k get svc -A | grep ingress
tanzu-system-ingress   contour                  ClusterIP      198.51.100.113   <none>         8001/TCP                     7m17s
tanzu-system-ingress   envoy                    LoadBalancer   198.51.100.151   192.168.5.44   80:30533/TCP,443:30113/TCP   7m17s
[root@orfdns contour]# 
```

![dns_start_record](https://github.com/ogelbric/contour_envoy_DNS_star_record/blob/main/dns_star_record.png)

## Deploy test namespace
```
[root@orfdns envoy_contour]# k apply -f ./myapp-ingress-namespace.yaml
namespace/myapp created
[root@orfdns envoy_contour]# cat myapp-ingress-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2021-05-14T14:33:04Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:status:
        f:phase: {}
    manager: kubectl
    operation: Update
    time: "2021-05-14T14:33:04Z"
  name: myapp
  resourceVersion: "1059400"
  selfLink: /api/v1/namespaces/my-ingress-app
  uid: d33a1558-2bbd-4cff-aab9-1180e84c04ed
spec:
  finalizers:
  - kubernetes
status:
```

## Deploy the ingress services 
```
[root@orfdns envoy_contour]# k apply -f ./myapp-ingress-services.yaml
service/pacman created
ingress.networking.k8s.io/pacman created
[root@orfdns envoy_contour]# cat myapp-ingress-services.yaml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    name: pacman
  name: pacman
  namespace: myapp
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    name: pacman
  sessionAffinity: None
  type: ClusterIP  # please note the type here from NodePort to ClusterIP and the different spec.ports.* section
status:
---
apiVersion: networking.k8s.io/v1
kind: Ingress # Orf, we introduce a an ingress Object now that helps us route things
metadata:
  name: pacman
  namespace: myapp
spec:
  rules:
  - host: pacman.apps.lab.local # for this example, this is *.apps.lab.local is my wildcard DNS entry that points to the VIP in the NSX-T LB i.e. 192.168.5.44
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: pacman
            port:
              number: 80	      
```

## Check in ingress
```
[root@orfdns envoy_contour]# k get ingress -n myapp
NAME     CLASS    HOSTS                   ADDRESS        PORTS   AGE
pacman   <none>   pacman.apps.lab.local   192.168.5.44   80      104s
[root@orfdns envoy_contour]# 
```

## Deploy a test app
```
[root@orfdns envoy_contour]# k apply -f ./myapp-app.yaml 
deployment.apps/pacman created
[root@orfdns envoy_contour]# cat myapp-app.yaml 
---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
  labels:
    name: pacman
  name: pacman
  namespace: myapp
spec:
  selector:
    matchLabels:
      name: pacman
  replicas: 1
  template:
    metadata:
      labels:
        name: pacman
    spec:
      containers:
      - name: pacman
        image: gcr.io/boreal-rain-xxxxxx/nginx #replace with nginx from docker or your own repo
        ports:
        - containerPort: 80
 ```
 
 ## Test the app
 ```
 [root@orfdns envoy_contour]# curl pacman.apps.lab.local
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>

```


![test_app](https://github.com/ogelbric/contour_envoy_DNS_star_record/blob/main/test_app.png)


## Deploy test app number 2
```
[root@orfdns envoy_contour]# k apply -f ./newapp.yaml 
deployment.apps/pacwoman created
service/pacwoman created
ingress.networking.k8s.io/pacwoman created
[root@orfdns envoy_contour]# k get ingress -n myapp
NAME       CLASS    HOSTS                      ADDRESS        PORTS   AGE
pacman     <none>   pacman.apps.lab.local      192.168.5.44   80      20m
pacwoman   <none>   pacwoman2.apps.lab.local   192.168.5.44   80      8s
[root@orfdns envoy_contour]# cat newapp.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
  labels:
    name: pacwoman
  name: pacwoman
  namespace: myapp
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      name: pacwoman
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        name: pacwoman
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: beta.kubernetes.io/os
                operator: In
                values:
                - linux
      containers:
      - image: gcr.io/boreal-rain-256712/nginx
        imagePullPolicy: IfNotPresent
        name: pacwoman
        ports:
        - containerPort: 80
          name: http-server
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
---
apiVersion: v1
kind: Service
metadata:
  labels:
    name: pacwoman
  name: pacwoman
  namespace: myapp
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    name: pacwoman
  sessionAffinity: None
  type: ClusterIP
status:
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pacwoman
  namespace: myapp
spec:
  rules:
  - host: pacwoman2.apps.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: pacwoman
            port:
              number: 80
```


## Reference documentation
```
Doc: https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-FA305850-ED37-48C6-99E9-5B6996376F0E.html
Install TKG extensions pre-requs
https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-36C7D7CB-312F-49B6-B542-1D0DBC550198.html

```
