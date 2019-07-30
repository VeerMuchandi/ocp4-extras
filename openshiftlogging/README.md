# Installing OpenShift Logging on OCP4

## Install Elasticsearch operator 

* From Operator Hub choose `Elasticsearch` operator from `Logging and Tracing`
* Wait until upgrade status is `Up to date` and  `1 installed`. You will observe the operator pod running.

```
$ oc get po -n openshift-operators
NAME                                       READY   STATUS    RESTARTS   AGE
elasticsearch-operator-7c5cc4bff9-pf8lb    1/1     Running   0          3m7s
```

## Install OpenShift Cluster Logging

**Note :** Tested with the Cluster Logging Operator version 4.1.4

* Goto  `Administration`->`Namespaces` and `Create` a namespace with name `openshift-logging` with following labels

```
openshift.io/cluster-logging=true 
openshift.io/cluster-monitoring=true 
```

* Install openshift cluster logging using `cluster logging operator` from Operator Hub
    *  Choose  `openshift-logging` that we created earlier as the  specific namespace on the cluster
    * Approval strategy `Automatic`

* Wait until upgrade status is `Up to date` and  `1 installed`. You will observe the operator pod running.

```
oc get po -n openshift-logging
NAME                                       READY   STATUS    RESTARTS   AGE
cluster-logging-operator-8cb69d977-sqmm2   1/1     Running   0          6m26s
```

* Now click on the link `1 installed`  and  `Create New` for `Cluster Logging` 

* Edit the yaml to include the following resources requirements. The default will configure the Elasticsearch pods to require `16Gi` memory. Instead we are configuring it to use lower memory as below. Add these resource requirements right before `storage:` for elasticsearch

```
      resources:
        limits:
          memory: 8Gi
        requests:
          cpu: 200m
          memory: 4Gi
      storage:
```

* Click on `Create` on the next screen

**Note :** In spite of making this change, only one of the elasticsearch pods came up and the other two were pending due to lack of resources. So I had to add additional nodes (on AWS, change the count on MachineSet and wait for a few mins to nodes to be added)


* Watch creation on  EFK pods

```
$ oc get po -n openshift-logging 
NAME                                            READY   STATUS    RESTARTS   AGE
cluster-logging-operator-5fbb755d9c-c5dv4       1/1     Running   0          3h39m
elasticsearch-cdm-w9sf54if-1-bcd8ffc87-5gxz2    2/2     Running   0          9m47s
elasticsearch-cdm-w9sf54if-2-54df86d48d-xkq5z   2/2     Running   0          9m39s
elasticsearch-cdm-w9sf54if-3-6c96bbd554-7kl66   2/2     Running   0          9m32s
fluentd-5stvp                                   1/1     Running   0          9m42s
fluentd-926tj                                   1/1     Running   0          9m42s
fluentd-hqgb4                                   1/1     Running   0          9m42s
fluentd-j8xhg                                   1/1     Running   0          9m42s
fluentd-l5ssm                                   1/1     Running   0          9m42s
fluentd-llbwk                                   1/1     Running   0          9m42s
fluentd-lz4w6                                   1/1     Running   0          9m42s
fluentd-x89cw                                   1/1     Running   0          9m42s
fluentd-xs74x                                   1/1     Running   0          9m42s
kibana-57ff8486f8-p7bk4                         2/2     Running   0          9m47s
```

Identify the route in the `openshift-logging` project

```
$ oc get route -n openshift-logging
NAME     HOST/PORT                                             PATH   SERVICES   PORT    TERMINATION          WILDCARD
kibana   kibana-openshift-logging.apps.YOURDOMAIN          kibana     <all>   reencrypt/Redirect   None

```

Access this url and login


## Cleanup

Delete the CR 
```
$ oc get clusterlogging -n openshift-logging
NAME       AGE
instance   55m

$ oc delete clusterlogging instance -n openshift-logging
clusterlogging.logging.openshift.io "instance" deleted
```

Remove  pvcs yourself

```
$ oc delete pvc -n openshift-logging --all
persistentvolumeclaim "elasticsearch-elasticsearch-cdm-aeslo1r7-1" deleted
persistentvolumeclaim "elasticsearch-elasticsearch-cdm-aeslo1r7-2" deleted
persistentvolumeclaim "elasticsearch-elasticsearch-cdm-aeslo1r7-3" deleted
```

Confirm that all the pods except the operator are  gone

```
$ oc get po
NAME                                        READY   STATUS    RESTARTS   AGE
cluster-logging-operator-5fbb755d9c-c5dv4   1/1     Running   0          4h26m
```

