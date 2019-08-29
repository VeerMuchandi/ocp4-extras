# Cleanup Hanging Objects

## Cleaning up projects hanging in `Terminating` State

**Use Case**: If you have a bunch of projects hanging around inspite of being deleted.

### Prereqs
* `jq` is used in the command below. I am running it from Mac
* Login as `admin` user

Run the following script:
```
for i in $( kubectl get ns | grep Terminating | awk '{print $1}'); do echo $i; kubectl get ns $i -o json| jq "del(.spec.finalizers[0])"> "$i.json"; curl -k -H "Authorization: Bearer $(oc whoami -t)" -H "Content-Type: application/json" -X PUT --data-binary @"$i.json" "$(oc config view --minify -o jsonpath='{.clusters[0].cluster.server}')/api/v1/namespaces/$i/finalize"; done
```


## Cleaning up Persistent Volumes hanging in `Terminating` State

**Use Case**: If you have a bunch of PVCs and PVs hanging around inspite of being released.

This script identifies all PVCs in `Terminating` state and removes the finalizers from their spec

```
for i in $(oc get pvc | grep Terminating| awk '{print $1}'); do oc patch pvc $i --type='json' -p='[{"op": "replace", "path": "/metadata/finalizers", "value":[]}]'; done
```

After running the above all hanging PVCs will be gone.

If you want to get rid of all hanging PVs run the following command after above.

```
for i in $(oc get pv | grep Released| awk '{print $1}'); do oc patch pv $i --type='json' -p='[{"op": "replace", "path": "/metadata/finalizers", "value":[]}]'; done

```
## Node running out of IP Addresses

**Use case:** Workloads (pods) get assigned to node but they stay in `Container Creating` mode. If you check project events, it shows an error that looks like this

```
Failed create pod sandbox: rpc error: code = Unknown desc = failed to create pod network sandbox k8s_mongodb-1-9qnpx_catalyst1_558c29e7-b78d-11e9-99d8-5254006dbdbc_0(0f93c64baabcce3f6a9396f106b3fe1e27611e4f74d74914e99ff7a063a8c6a7): Multus: Err adding pod to network "openshift-sdn": Multus: error in invoke Delegate add - "openshift-sdn": CNI request failed with status 400: 'failed to run IPAM for 0f93c64baabcce3f6a9396f106b3fe1e27611e4f74d74914e99ff7a063a8c6a7: failed to run CNI IPAM ADD: failed to allocate for range 0: no IP addresses available in range set: 10.254.5.1-10.254.5.254
```

However, if you see what is running on the node it shows nothing other than static pods.

```
$ oc get po -o wide --all-namespaces | grep Running | grep worker2
openshift-cluster-node-tuning-operator                  tuned-gcp5m                                                       1/1     Running             2          5h41m   192.168.1.46   worker2.ocp4.home.ocpcloud.com   <none>           <none>
openshift-machine-config-operator                       machine-config-daemon-9klh9                                       1/1     Running             2          5h41m   192.168.1.46   worker2.ocp4.home.ocpcloud.com   <none>           <none>
openshift-monitoring                                    node-exporter-jfxwq                                               2/2     Running             4          5h39m   192.168.1.46   worker2.ocp4.home.ocpcloud.com   <none>           <none>
openshift-multus                                        multus-lj8cq                                                      1/1     Running             2          5h41m   192.168.1.46   worker2.ocp4.home.ocpcloud.com   <none>           <none>
openshift-sdn                                           ovs-l9qt5                                                         1/1     Running             1          5h11m   192.168.1.46   worker2.ocp4.home.ocpcloud.com   <none>           <none>
openshift-sdn                                           sdn-lqss7                                                         1/1     Running             2          5h11m   192.168.1.46   worker2.ocp4.home.ocpcloud.com   <none>           <none>
```

On the other hand `oc get nodes` shows all nodes are `Ready`.

### What's happening?
Somehow the pod deletion is not complete on the nodes (may be when the node gets restarted), and the IP addresses allocated to the pods on that node are all listed as already used.

You'll need to SSH to the node to be able to get out of this situation

**Note:** OpenShift discourages SSHing to the node. But as of now I don't know any other way of dealing with this.

Login to the node and list the following location:

```
# ls /var/lib/cni/networks/openshift-sdn/ | wc -l

254
```

You will see there are a bunch of ip addresses here that aren't cleaned up.

### Quick fix

Make sure only static pods are running on that node. If you had restarted the node, and no pods are coming up on that node, then only static pods will be running anyway!!

Now remove all the files with IP address names in this location:

```
# rm /var/lib/cni/networks/openshift-sdn/10*
```

As soon as you do that the node should be running normally.


### Confirm node is normal

Try deleting a pod running on this node just to test. You should see an entry in CRIO logs `journalctl -u crio -l --no-pager`to this effect:

```
Aug 05 20:00:18 worker2.ocp4.home.ocpcloud.com crio[897]: 2019-08-05T20:00:18Z [verbose] Del: bluegreen:green-2-jbv56:openshift-sdn:eth0 {"cniVersion":"0.3.1","name":"openshift-sdn","type":"openshift-sdn"}
```

and the corresponding ip address of the pod (`oc get po -o wide` should show the PODIP) should be removed from this location `/var/lib/cni/networks/openshift-sdn`. If so, your node is normal again.










