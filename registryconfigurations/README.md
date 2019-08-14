# Configuring Insecure Registries, Blocking Registries and so on

## Prerequisites
You will need admin access to the cluster
Login as the admin user


## Understanding the images CRD

Read the following CRD for details of all the features that can be changed

```
$ oc get crds images.config.openshift.io -n openshift-config -o yaml
```

Also look at the CR created by default for this CRD. It should look like this

```
$ oc get images.config.openshift.io cluster -o yaml
apiVersion: config.openshift.io/v1
kind: Image
metadata:
  annotations:
    release.openshift.io/create-only: "true"
  creationTimestamp: "2019-07-09T19:30:12Z"
  generation: 1
  name: cluster
  resourceVersion: "985798"
  selfLink: /apis/config.openshift.io/v1/images/cluster
  uid: f83bf84a-a27f-11e9-ab36-5254006a091b
spec: {}
status:
  externalRegistryHostnames:
  - default-route-openshift-image-registry.apps.ocp4.example.mycloud.com
  internalRegistryHostname: image-registry.openshift-image-registry.svc:5000
```

The `spec:` above is where we can add all customizations.


A couple of examples are shown here:

## To block a registry

Let us say we want to block `docker.io`

Here is what we can do. Edit the CR

```
$ oc edit images.config.openshift.io cluster
```
Edit the spec section as follows

```
spec:
  registrySources:
    blockedRegistries:
    - docker.io
```

This change should update all the masters and nodes and add this change `[registries.block]
    registries = ["docker.io"]`


```
[core@worker0 ~]$ cat /etc/containers/registries.conf 
[registries]
  [registries.search]
    registries = ["registry.access.redhat.com", "docker.io"]
  [registries.insecure]
    registries = []
  [registries.block]
    registries = ["docker.io"]
```

Once all the nodes are up if you try to create an application using an image from docker.io, it will be blocked. You will see these kind of error messages in the events.

```

1s          Warning   InspectFailed       pod/welcome-1-fwcp4               Failed to inspect image "docker.io/veermuchandi/welcome@sha256:db4d49ca6ab825c013cb076a2824947c5378b845206cc29c6fd245744ffd35fb": rpc error: code = Unknown desc = cannot use "docker.io/veermuchandi/welcome@sha256:db4d49ca6ab825c013cb076a2824947c5378b845206cc29c6fd245744ffd35fb" because it's blocked
1s          Warning   Failed              pod/welcome-1-fwcp4               Error: ImageInspectError

```


## Configuring Insecure Registries

Update the CR `$ oc edit images.config.openshift.io cluster`

with the spec as follows

```
spec:
  registrySources:
    insecureRegistries:
    - bastion.mycloud.com:5000
    - 198.18.100.1:5000
```

This change should update all the masters and nodes and add this change

```
[core@worker-0 ~]$ sudo cat /etc/containers/registries.conf
[registries]
  [registries.search]
    registries = ["registry.access.redhat.com", "docker.io"]
  [registries.insecure]
    registries = ["bastion.mycloud.com:5000", "198.18.100.1:5000"]
  [registries.block]
    registries = []
```






