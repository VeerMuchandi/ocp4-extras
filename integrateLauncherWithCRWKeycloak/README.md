# Integrating Launcher with Keycloak installed by CRW

## Install CodeReady Workspaces
Work in progress

## Install Fabric8 Launcher

### Deploy Launcher Operator

> **NOTE:** Launcher Operator is not yet part of OperatorHub. So we have to deploy launcher operator manually

Clone the launcher code from its git repo
```
$ git clone https://github.com/fabric8-launcher/launcher-operator
$ cd launcher-operator
```

Create a new project on your openshift cluster for launcher

```
$ oc new-project launcher-infra
```

Deploy Launcher Operator

```
$ oc create -R -f ./deploy

customresourcedefinition.apiextensions.k8s.io/launchers.launcher.fabric8.io created
deployment.apps/launcher-operator created
role.rbac.authorization.k8s.io/launcher-operator created
rolebinding.rbac.authorization.k8s.io/launcher-operator created
serviceaccount/launcher-operator created
```

As you can see above, it deploys CRD, role, role binding, service account, and deployment for launcher operator.

In a few mins you should see launcher operator running

```
$ oc get po
NAME                                 READY   STATUS    RESTARTS   AGE
launcher-operator-69ddb677dd-c72jw   1/1     Running   0          3m23s
```

### Install Launcher on OpenShift

Launcher operator template custom resources (CRs) are available in the example folder. Edit the CR for your cluster. Mine looks like this.

The keycloak that I am using here was installed earlier with EclipseChe 7.1. That keycloak instance already integrates with openshift-v4 and will authenticate against your openshift environment. Using that keycloak with launcher will allow your launcher to automatically authenticate against your openshift cluster.

```
$ cat example/launcher_cr.yaml
apiVersion: launcher.fabric8.io/v1alpha2
kind: Launcher
metadata:
  name: launcher
spec:

####### OpenShift Configuration
  openshift:
    consoleUrl: https://console-openshift-console.apps.ocp4.home.ocpcloud.com
    apiUrl: https://openshift.default.svc.cluster.local

####### OAuth Configuration
  oauth:
    enabled: true
    url: https://oauth-openshift.apps.ocp4.home.ocpcloud.com
    keycloakUrl: https://keycloak-che71.apps.ocp4.home.ocpcloud.com/auth
    keycloakRealm: che
    keycloakClientId: che-public
```

In the above CR
* **consoleURL** is your OCP web console URL
* **apiURL** This is your OCP4 cluster's API URL. You can leave the value as `https://openshift.default.svc.cluster.local` if you are using the same cluster where the launcher is running
* **oauth.url** is the oauth URL for your cluster. You can run `oc get route -n openshift-authentication` to get this value for your cluster.
* **keycloakURL** is the route for your keycloak server. In this case, I am using the keycloak installed by EclipseChe operator. 
* **keyCloakRealm** is the realm configured in your KeyCloakServer. Login to your Keycloak Administrator Console and look at `Realm Settings` to find `Name` on `General`tab
* **keycloakClientId** is the client to be used on Keycloak Server. In this case created by Che installation. You will find this in the `Clients` section when you login to keycloak admin console


Create the custom resource now. Operator will create a launcher once it finds this custom resource

```
$ oc create -f example/launcher_cr.yaml
launcher.launcher.fabric8.io/launcher created
```

You will see launcher pod running in a few mins

```
$ oc get po
NAME                                 READY   STATUS      RESTARTS   AGE
launcher-application-1-deploy        0/1     Completed   0          5d2h
launcher-application-1-qn4l2         1/1     Running     0          51m
launcher-operator-69ddb677dd-x69xs   1/1     Running     0          5d2h
```

Find launcher URL by inquiring on the route.
```
$ oc get route launcher --template={{.spec.host}}
launcher-launcher-infra.apps.ocp4.home.ocpcloud.com
```
Use this to log on to your launcher.


