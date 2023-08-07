# Introduction to Custom Metrics Autoscaling

OpenShift comes with two Pod autoscaler's built in, the [Horizontal Pod Autoscaler](https://docs.openshift.com/container-platform/4.13/nodes/pods/nodes-pods-autoscaling.html) and the [Vertical Pod Autoscaler](https://docs.openshift.com/container-platform/4.13/nodes/pods/nodes-pods-vertical-autoscaler.html). The Horizontal Pod Autoscaler (HPA) is used to increase and decrease the number of pods that are running based on either CPU usage or Memory Usage. The Vertical Pod Autoscaler (VPA) is designed to find the optimal settings for the requests and limits on a pod or deployment config. 

What if your application doesn't scale based on CPU usage or Memory usage? What if you want to scale based on an application queue, or backlog. Perhaps you need to scale your application based on the number of requests are occurring. These tasks can be handled by the [Custom Metrics Autoscaler Operator](https://docs.openshift.com/container-platform/4.13/nodes/cma/nodes-cma-autoscaling-custom.html) which is based on the upstream [Keda](https://keda.sh/) Project. 

The Custom Metrics Autoscaler (CMA) makes use of metrics gathered by Prometheus and queries Prometheus to get the current state of of your application. The CMA can then scale your application (through the use of Kuberenetes Deployments, Statefulsets, or any Custom Resource that defines a "/scale" subresource) both up and down based on your application load.

> **NOTE:** For the Custom Metrics Autoscaler to work with your application, your application needs to export its metrics in the [Prometheus exposition format](https://github.com/prometheus/docs/blob/main/content/docs/instrumenting/exposition_formats.md) or [OpenMetrics](https://github.com/OpenObservability/OpenMetrics) format.

In order to demo this Custom Metrics AutoScaling feature we will leverage the OpenShift [User Application Monitoring](https://docs.openshift.com/container-platform/4.13/monitoring/enabling-monitoring-for-user-defined-projects.html) feature so that we can define our monitoring endpoints.

## Install User Application Monitoring

Let's start by enabling User Application Monitoring in OpenShift:

1. Visit the OpenShift web console
2. Log in as a user with cluster-admin privileges
3. Make sure to select the Administrator perspective in the upper left
4. Click Workloads and then Config Maps
5. Ensure that the "openshift-monitoring" project is selected
6. Click Create Config Map
7. Paste the following YAML into the box:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
```

Navigate to the openshift-user-workload-monitoring project ... You will see a Prometheus monitoring stack (Prometheus and Thanos) starting up. 

## Install Custom Metrics AutoScaler

With *User Application Monitoring* enabled, we will now install the Custom Metrics AutoScaler Operator, and start an instance of the Keda Controller. To start, install the *Custom Metrics AutoScaler*:

1. In the OpenShift Container Platform web console, click Operators → OperatorHub.
2. Choose Custom Metrics Autoscaler from the list of available Operators, and click Install.
3. On the Install Operator page, ensure that the All namespaces on the cluster (default) option is selected for Installation Mode. This installs the Operator in all namespaces.
4. Ensure that the openshift-keda namespace is selected for Installed Namespace. OpenShift Container Platform creates the namespace, if not present in your cluster.
5. Click Install.
6. Verify the installation by listing the Custom Metrics Autoscaler Operator components:
    a. Navigate to Workloads → Pods.
    b. Select the openshift-keda project from the drop-down menu and verify that the custom-metrics-autoscaler-operator-* pod is running.
    c. Navigate to Workloads → Deployments to verify that the custom-metrics-autoscaler-operator deployment is running.

### Install the Keda Controller

Now, create an instance of the Keda Controller:

1. In the OpenShift Container Platform web console, click Operators → Installed Operators.
2. Click Custom Metrics Autoscaler.
3. On the Operator Details page, click the KedaController tab.
4. On the KedaController tab, click Create KedaController and edit the file.

```yaml
kind: KedaController
apiVersion: keda.sh/v1alpha1
metadata:
  name: keda
  namespace: openshift-keda
spec:
  admissionWebhooks:
    logEncoder: console
    logLevel: info
  metricsServer:
    logLevel: '0'
    auditConfig: 
      logFormat: "json"
      lifetime:
        maxAge: "2"
        maxBackup: "1"
        maxSize: "50"    
  operator:
    logEncoder: console
    logLevel: info
  serviceAccount: null
  watchNamespace: ''
```

5. Click Create KEDAController

## Deploy Test Application

We will be using [k8s memuser advanced](https://github.com/xphyr/k8s_memuser_advanced) which is a simple demo application that can be used to test things like pod Autoscaling. It exports metrics in Prometheus format and includes the following application specific metrics:

```
# HELP http_requests_total Count of all HTTP requests
# TYPE http_requests_total counter
http_requests_total{function="HelloServer"} 1
# HELP myapp_processed_ops_total The total number of simulated processed ops.
# TYPE myapp_processed_ops_total counter
myapp_processed_ops_total 1995
```

You can clone the k8s memuser advanced repo above and build the application yourself, or you can use the included deployment files 

```sh
$ oc create new-project cmstest
Now using project "cmstest" 
$ oc create -f cms-deployment.yaml
service/memuser-svc created
deployment.apps/mem-user created
$ oc expose svc/memuser-svc
route.route.openshift.io/memuser-svc exposed
$ oc get route
NAME          HOST/PORT                                   PATH   SERVICES      PORT   TERMINATION   WILDCARD
memuser-svc   memuser-svc-cmstest.apps.test.example.com          memuser-svc   8080                 None
```

Test that the application is available by running curl against the "HOST/PORT" from the output above.

```sh
$ curl http://memuser-svc-cmstest.apps.test.example.com
Hello User. My current memory usage is:
 Alloc = 2 MiB	Sys = 20 MiB	NumGC = 100
```

You can also view the Prometheus stats by querying the metrics endpoint:

```sh
$ curl http://memuser-svc-cmstest.apps.test.example.com/metrics
...
# HELP http_requests_total Count of all HTTP requests
# TYPE http_requests_total counter
http_requests_total{function="HelloServer"} 7
# HELP myapp_processed_ops_total The total number of simulated processed ops.
# TYPE myapp_processed_ops_total counter
myapp_processed_ops_total 4928
# HELP version Version information about this binary
# TYPE version gauge
version{version="(devel)"} 0
```

## Create Service Monitor

With our application up and running, its time to start monitoring it with User Application Monitoring. Create a file with the YAML listed below:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: memuser-monitor
 spec:
   endpoints:
   - interval: 30s
     port: web
     scheme: http
   selector:
     matchLabels:
       app: memuser
```

> **NOTE:** The selector in the above yaml is matching against a service, so ensure your service is properly labeled.

Now apply the ServiceMonitor to your cluster:

```sh
$ oc create -f serviceMonitor.yml
```

We now have an application deployed in OpenShift and we are monitoring ith with Prometheus. Its now time to create a KEDA ScaledObject.

## Creating a ScaledObject

In order to create a ScaledObject using the built in OpenShift Prometheus service, we need three things:

* A service account to authenticate to Prometheus with
* A TriggerAuthentication Object
* A ScaledObject 

### Create a ServiceAccount and associated Role

Start by creating a service account in the `cmstest` Namespace:

```sh
$ oc project cmstest
$ oc create serviceaccount kedasa
$ oc describe sa/kedasa
Name:                kedasa
Namespace:           cmstest
Labels:              <none>
Annotations:         <none>
Image pull secrets:  kedasa-dockercfg-f8vsz
Mountable secrets:   kedasa-dockercfg-f8vsz
Tokens:              kedasa-token-2gltj
Events:              <none>
```

We now need to create a role in your Namespace that can be assigned to the serviceAccount we created above. Create a `thanos-metrics-reader.yml` file with the following contents:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: thanos-metrics-reader
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - metrics.k8s.io
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
```

Now create a `roleBinding.yml` file with the following contents:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: thanos-metrics-reader 
  namespace: cmstest 
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: thanos-metrics-reader
subjects:
- kind: ServiceAccount
  name: kedasa 
  namespace: cmstest 
```

We will now apply the new role, and then give that role to our SA account

```sh
$ oc create -f thanos-metrics-reader.yaml
role.rbac.authorization.k8s.io/thanos-metrics-reader created
$ oc create -f roleBinding.yml
```

### Create TriggerAuthentication

A TriggerAuthentication object is required to allow the Keda controller to access the Prometheus data on your behalf. It will leverage the ServiceAccount created in the previous step. Create a `triggerAuth.yml` file with the following information:

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth-prometheus
spec:
  secretTargetRef: 
  - parameter: bearerToken 
    name: kedasa-token-2gltj 
    key: token 
  - parameter: ca
    name: kedasa-token-2gltj
    key: ca.crt
```

Apply the file to your cluster:

```sh
$ oc create -f triggerAuth.yml
```

## Create ScaledObject Definition

We can now create a ScaledObject definition in our cluster. The following YAML will create a ScaledObject that will scale up to a Maximum of 10 replicas with a minimum of 1 replica, and it will scale up as long as the rate of http_requests is greater than 5.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: prom-scaledobject-memuser
  namespace: cmstest
spec:
  scaleTargetRef:
    apiVersion: apps/v1 
    name: mem-user 
    kind: Deployment 
  cooldownPeriod:  200 
  maxReplicaCount: 10 
  minReplicaCount: 1
  fallback: 
    failureThreshold: 3
    replicas: 2
  pollingInterval: 30 
  advanced:
    restoreToOriginalReplicaCount: false 
  triggers:
    - type: prometheus 
      metadata:
        serverAddress: https://thanos-querier.openshift-monitoring.svc.cluster.local:9092 
        namespace: cmstest 
        metricName: http_requests_total 
        threshold: '5' 
        query: sum(rate(http_requests_total{job="memuser-svc"}[1m])) 
        authModes: "bearer"
      authenticationRef: 
        name: keda-trigger-auth-prometheus
```

We now need a way to increase the rate of traffic going to our application. For this we will use [siege](https://github.com/JoeDog/siege). Run `siege` against the URL for the application and leave it running. This will create traffic to the test site, and start to drive up HTTP requests.

```sh
$ siege http://memuser-svc-cmstest.apps.test.example.com
```

Siege will generate a large number of requests against the application, and drive up the http_requests_total counter. This will then trigger the Custom Metrics Autoscaler to scale the memuser deployment up to meet the demand. You can watch this occur:

```sh
$ oc get hpa --watch
NAME                                 REFERENCE             TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-prom-scaledobject-memuser   Deployment/mem-user   0/5 (avg)   1         10        1          5d6h
keda-hpa-prom-scaledobject-memuser   Deployment/mem-user   0/5 (avg)   1         10        1          5d6h
keda-hpa-prom-scaledobject-memuser   Deployment/mem-user   0/5 (avg)   1         10        1          5d6h
keda-hpa-prom-scaledobject-memuser   Deployment/mem-user   57260m/5 (avg)   1         10        1          5d6h
keda-hpa-prom-scaledobject-memuser   Deployment/mem-user   66561m/5 (avg)   1         10        4          5d6h
keda-hpa-prom-scaledobject-memuser   Deployment/mem-user   60445m/5 (avg)   1         10        8          5d6h
keda-hpa-prom-scaledobject-memuser   Deployment/mem-user   66605m/5 (avg)   1         10        10         5d6h
keda-hpa-prom-scaledobject-memuser   Deployment/mem-user   64289m/5 (avg)   1         10        10         5d6h
```

Note that as time goes on the number of Replicas continues to increase but Maxes out at 10. If you stop the `siege` process, and wait, you will see the number of Replicas start to decrease.

```sh
$ oc get hpa --watch
NAME                                 REFERENCE             TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-prom-scaledobject-memuser   Deployment/mem-user   0/5 (avg)        1         10        10         5d6h
keda-hpa-prom-scaledobject-memuser   Deployment/mem-user   0/5 (avg)        1         10        10         5d6h
keda-hpa-prom-scaledobject-memuser   Deployment/mem-user   0/5 (avg)        1         10        1          5d6h
```

## References:

[](https://docs.openshift.com/container-platform/4.12/nodes/cma/nodes-cma-autoscaling-custom.html)