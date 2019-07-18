# Adding NFS Auto Provisioner to your cluster

https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client

# oc new-project nfsprovisioner
# NS=$(kubectl config get-contexts|grep -e "^\*" |awk '{print $5}')
# echo $NS
nfsprovisioner
# NAMESPACE=${NS:-default}

# mkdir nfsprovisioner
# cd nfsprovisioner/
# mkdir deploy


[root@veerocp4helper nfsprovisioner]# ls -R
.:
deploy  deployment.yaml

./deploy:
class.yaml  deployment.yaml  rbac.yaml



# sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" ./deploy/rbac.yaml

# cat deploy/rbac.yaml 
kind: ServiceAccount
apiVersion: v1
metadata:
  name: nfs-client-provisioner
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: nfsprovisioner
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: nfsprovisioner
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io



# kubectl create -f deploy/rbac.yaml 
serviceaccount/nfs-client-provisioner created
clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created

# oc adm policy add-scc-to-user hostmount-anyuid system:serviceaccount:$NAMESPACE:nfs-client-provisioner
securitycontextconstraints.security.openshift.io/hostmount-anyuid added to: ["system:serviceaccount:nfsprovisioner:nfs-client-provisioner"]

# wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/deployment.yaml -O deploy/deployment.yaml

# cat deploy/deployment.yaml 
apiVersion: v1
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: nfs-provisioner
            - name: NFS_SERVER
              value: 192.168.1.50
            - name: NFS_PATH
              value: /exports         
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.1.50
            path: /exports         


# cat deploy/class.yaml 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: nfs-provisioner # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false" # When set to "false" your PVs will not be archived
                           # by the provisioner upon deletion of the PVC.
                           

# oc create -f deploy/deployment.yaml 

# oc create -f deploy/class.yaml 

# oc annotate storageclass managed-nfs-storage storageclass.kubernetes.io/is-default-class="true"
storageclass.storage.k8s.io/managed-nfs-storage annotated