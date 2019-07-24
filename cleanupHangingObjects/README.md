# Cleanup Hanging Objects

## Cleaning up projects hanging in `Terminating` State

**Use Case**: If you have a bunch of projects hanging around inspite of being deleted.

### Prereqs
* `jq` is used in the command below. I am running it from Mac
* Login as `admin` user

Set the value of your API server

```
export APIURL=https://api.ocp4.home.mycloud.com:6443
```

Run the following script:
```
for i in $( kubectl get ns | grep Terminating | awk '{print $1}'); do echo $i; kubectl get ns $i -o json| jq "del(.spec.finalizers[0])"> "$i.json"; curl -k -H "Authorization: Bearer $(oc whoami -t)" -H "Content-Type: application/json" -X PUT --data-binary @"$i.json" "$APIURL/api/v1/namespaces/$i/finalize"; done
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
