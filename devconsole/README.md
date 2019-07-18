# Install the latest Dev Console as another application (to test)

**Acknowledgements:** Based on Steve Speicher's https://github.com/sspeiche/dev-console

## Pre-requisites
* OCP4 Cluster
* You will need admin access and login as admin user to do this


## Steps

1. Clone this repo and change to `devconsole`

2. Create a new project
```
oc new-project devconsole
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

Update the base64 encoded value of your secret in `console-oauth-config.yaml` (`echo <<secretOfYourChoice>> | base64`).

Now create console-oauth-config
```
oc create -f console-oauth-config.yaml
```

5. Deploy the new Developer Console using the template

```
oc process -f devconsole-template.json -p OPENSHIFT_API_URL=<<your api url along with protocol and port>> -p OPENSHIFT_CONSOLE_URL=<<your new devconsole url without https>> | oc create -f -
```
