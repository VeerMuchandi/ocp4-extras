# Install the latest Dev Console as another application (to test)

**Acknowledgements:** Based on Steve Speicher's https://github.com/sspeiche/dev-console

## Pre-requisites
* OCP4 Cluster
* You will need admin access and login as admin user to do this


## Steps

### Short Script

If you want don't want to follow step by step instructions in details. Just run the following script with your own value for `SECRET`

```
###############################
# INSTALL DEVCONSOLE          #
###############################
export SECRET=mysecret
oc create namespace openshift-devconsole
oc project openshift-devconsole
oc get cm service-ca -n openshift-console --export -o yaml | sed -e "s/app: console/app: devconsole/g" | oc apply -n openshift-devconsole -f -
oc get secret console-serving-cert -n openshift-console --export -o yaml | oc apply -n openshift-devconsole -f -
oc get oauthclient console --export -o yaml | sed -e "s/name: console/name: devconsole/g" -e "s/console-openshift-console/devconsole/g" -e "s/^secret: .*/secret: $SECRET/g" | oc apply -f - 
curl https://raw.githubusercontent.com/VeerMuchandi/ocp4-extras/master/devconsole/console-oauth-config.yaml | sed -e "s/yourBase64EncodedSecret/$(echo -n $SECRET | base64)/g"| oc apply -f -
oc create clusterrole console --verb=get,list,watch --resource=customresourcedefinitions.apiextensions.k8s.io
oc create sa console -n openshift-devconsole
oc adm policy add-cluster-role-to-user console -z console -n openshift-devconsole
oc process -f https://raw.githubusercontent.com/VeerMuchandi/ocp4-extras/master/devconsole/devconsole-template.json -p OPENSHIFT_API_URL=$(oc cluster-info | awk '/Kubernetes/{printf $NF}'| sed -e "s,$(printf '\033')\\[[0-9;]*[a-zA-Z],,g") -p OPENSHIFT_CONSOLE_URL=$(oc get oauthclient devconsole -o jsonpath='{.redirectURIs[0]}'| sed -e 's~http[s]*://~~g' -e "s/com.*/com/g") | oc create -f - 
```

### Step by Step Instructions

1. Clone this repo and change to `devconsole`

2. Create a new namespace
**Note:** You won't be allowed to create a project that starts with `openshift-`. Hence we are creating a namespace. The template we process in the last step has a `priorityClass` annotation that requires an `openshift-*` namespace. Switch the project to `openshift-devconsole` after creating the namespace.

```
oc create namespace openshift-devconsole
oc project openshift-devconsole
```

3. Update service-ca-configmap.yaml, console-serving-cert.yaml with your values for similar objects from openshift-console project

Get the `service-ca` used by your console.
```
oc get cm service-ca -n openshift-console -o yaml
```
Copy the `data` section and update `service-ca-configmap.yaml`

```
oc create -f service-ca-configmap.yaml 
```

Get the serving certficate used by your console.
```
$ oc get secret console-serving-cert -n openshift-console -o yaml
```

Note the values of `data` section update the data sectio in `console-serving-cert.yaml` file.

```
oc create -f console-serving-cert.yaml 
```

4. Create an oauthclient for your new devconsole.

```
oc get oauthclient console -o yaml > oauthclient.yaml
```

Edit the values to look like below

```
$ cat oauthclient.yaml 
apiVersion: oauth.openshift.io/v1
grantMethod: auto
kind: OAuthClient
metadata:
  name: devconsole
redirectURIs:
- https://<<yourNewDevConsoleURL>>/auth/callback
secret: <<secretOfYourChoice>>
```
Note `clientId` is `devconsole` and the secret will be a string of your choice. You can leave the avlue you copied from the other `oauthclient` if you don't want to change it

Create oauthclient.
```
oc create -f oauthclient.yaml
```

Update the base64 encoded value of your secret in `console-oauth-config.yaml` (`echo -n <<secretOfYourChoice>> | base64`).

**Note: ** Make sure you use `echo -n`, otherwise it will add a newline and the secret won't work.

Now create console-oauth-config
```
oc create -f console-oauth-config.yaml
```

5. Deploy the new Developer Console using the template

```
oc process -f devconsole-template.json -p OPENSHIFT_API_URL=<<your api url along with protocol and port>> -p OPENSHIFT_CONSOLE_URL=<<your new devconsole url without https>> | oc create -f -
```
