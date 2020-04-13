# Adding NFS Auto Provisioner to your cluster

**Acknowledgements: ** Based on
https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client

## Prerequisites
* [NFS Server is set up and available](#appendix)
* OCP 4 cluster is setup and available. Admin access to this cluster

## Steps

1. Create a new project

```
# oc new-project nfsprovisioner
# NS=$(kubectl config get-contexts|grep -e "^\*" |awk '{print $5}')
# echo $NS
nfsprovisioner
# NAMESPACE=${NS:-default}
```

2. Set up folder structure like below
```
# mkdir nfsprovisioner
# cd nfsprovisioner/
# mkdir deploy
```

3. Get the needed files

```
wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/deployment.yaml -O deploy/deployment.yaml

wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/rbac.yaml -O deploy/rbac.yaml

wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/class.yaml -O deploy/class.yaml
```

Verify

```
[root@veerocp4helper nfsprovisioner]# ls -R
.:
deploy  

./deploy:
class.yaml  deployment.yaml  rbac.yaml
```

Update namespace

```
# sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" ./deploy/rbac.yaml
```

```
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
```

Add objects to cluster
```
# kubectl create -f deploy/rbac.yaml 
serviceaccount/nfs-client-provisioner created
clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
```

Elevate SCC to `hostmount-anyuid`

**If you don't have a cluster role** that points to `hostmount-anyuid` create one by running:

```
oc create clusterrole hostmount-anyuid-role --verb=use --resource=scc --resource-name=hostmount-anyuid
```
Now add the service account `nfs-client-provisioner` to that role by running

```
oc adm policy add-cluster-role-to-user hostmount-anyuid-role -z nfs-client-provisioner -n $NAMESPACE
```

The above method replaces the older way of editing default SCC i.e, `oc adm policy add-scc-to-user hostmount-anyuid system:serviceaccount:$NAMESPACE:nfs-client-provisioner` as this is discouraged in the openshift documentation now.


Edit `deploy/deployment.yaml` for the values of NFS server and NFS path. Both in `env` and `volumes`

```
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
              value: 199.168.2.150
            - name: NFS_PATH
              value: /exports         
      volumes:
        - name: nfs-client-root
          nfs:
            server: 199.168.2.150
            path: /exports         
```


Edit the `provisioner` name in `deploy/class.yaml`

```
# cat deploy/class.yaml 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: nfs-provisioner # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false" # When set to "false" your PVs will not be archived
                           # by the provisioner upon deletion of the PVC.
```

Deploy nfs-provisioner and storage-class

```
# oc create -f deploy/deployment.yaml 

# oc create -f deploy/class.yaml 
```


If you want this to be the default storage class, annotate the strae class accordingly

```
# oc annotate storageclass managed-nfs-storage storageclass.kubernetes.io/is-default-class="true"

storageclass.storage.k8s.io/managed-nfs-storage annotated
```

## <a name="appendix"></a> Appendix: NFS Setup 

```
# pvcreate /dev/vdb

# pvs
  PV         VG                  Fmt  Attr PSize   PFree  
  /dev/vda2  rhel_veerocp4helper lvm2 a--  <29.00g      0 
  /dev/vdb                       lvm2 ---  100.00g 100.00g
  
# vgcreate vg-nfs /dev/vdb
  Volume group "vg-nfs" successfully created
  
# lvcreate -n lv-nfs -l 100%FREE vg-nfs
  Logical volume "lv-nfs" created.

# lvs
  LV     VG                  Attr       LSize    Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root   rhel_veerocp4helper -wi-ao----  <26.00g                                                    
  swap   rhel_veerocp4helper -wi-ao----    3.00g                                                    
  lv-nfs vg-nfs              -wi-a----- <100.00g 
  
# mkfs.xfs /dev/vg-nfs/lv-nfs 
meta-data=/dev/vg-nfs/lv-nfs     isize=512    agcount=4, agsize=6553344 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=26213376, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=12799, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

# mkdir /exports

# vi /etc/fstab

# mount -a

# df -h
Filesystem                            Size  Used Avail Use% Mounted on
/dev/mapper/rhel_veerocp4helper-root   26G  4.3G   22G  17% /
devtmpfs                              1.9G     0  1.9G   0% /dev
tmpfs                                 1.9G   84K  1.9G   1% /dev/shm
tmpfs                                 1.9G  8.8M  1.9G   1% /run
tmpfs                                 1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/vda1                            1014M  145M  870M  15% /boot
tmpfs                                 379M     0  379M   0% /run/user/1000
/dev/mapper/vg--nfs-lv--nfs           100G   33M  100G   1% /exports

# yum install nfs-utils

# cat /etc/exports.d/veer.exports 
/exports *(rw,root_squash)

# exportfs
/exports 	<world>

# systemctl restart nfs

# showmount -e 199.168.2.150
Export list for 199.168.2.150:
/exports *

chown nfsnobody:nfsnobody /exports
chmod 777 /exports

firewall-cmd --add-service=nfs --permanent
firewall-cmd --add-service=nfs
```