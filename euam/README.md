# End-User Application Monitoring

OpenShift is a "batteries included" platform as a service. One of those "batteries" is the built in monitoring of memory, CPU, network, and disk usage of the OpenShift platform with the Prometheus, AlertManager and Grafana stack. These tools come pre-installed and configured to properly monitor the OpenShift platform and are integrated directly into the OpenShift console. But what if you want to monitor your application. OpenShift has you covered there as well, you can see the "speeds and feeds" of your application, including its memory, cpu disk and network usage out of the box, and is also integrated into the console.

But what if you want to take this farther? What if you want to monitor and alert on the internals of your application? Perhaps you want to monitor/observe the number of transactions that it is processing, or the number of "widgets" are being created, and what the latency of that "widget" creation is. You can instrument your application with custom metrics using the [OpenMetrics](https://openmetrics.io/) format. OpenMetrics builds on the Prometheus exporter format and can be ingested by Prometheus servers.

OpenShift now supports the ability to ingest these end user application metrics, hosted in OpenShift and make them available to view within the OpenShift console. The demo outlined below shows a **VERY** simple application that says "hello world", and also returns a 404 if you access the "/err" endpoint. The purpose of this application is to quickly show how OpenShift can ingest these user application metrics and even create alerts based on these metrics.

## Enable End-User Application Monitoring

End-user application monitoring is not enabled by default in OpenShift. It is a simple process to turn it on. Follow these steps to get End User Application Metrics enabled in your OpenShift cluster:

1. Visit the OpenShift web console
2. Log in as a user with cluster-admin privileges
3. Make sure to select the Administrator perspective in the upper left
4. Click Workloads and then Config Maps
5. Click Create Config Map
6. Paste the following YAML into the box:

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

At this point, the cluster is set up to create end user application monitoring, but it will not work for just anyone. You need to give additional permissions to users to enable this feature. If you are running this demo as a cluster-admin, you can do these next steps without issue. If you wish to enable users to create these configurations, follow the instructions here [Granting users permission to monitor user defined projects](https://docs.openshift.com/container-platform/4.6/monitoring/enabling-monitoring-for-user-defined-projects.html#granting-users-permission-to-monitor-user-defined-projects_enabling-monitoring-for-user-defined-projects)

## Deploy an Example Application

In order to take advantage of user application monitoring and more specific introspection you will need to add a Prometheus library to your application, or write your own library that writes the openmetrics exposition format. A list of libraries available for Prometheus are here: https://prometheus.io/docs/instrumenting/clientlibs/. To demonstrate this functionality we will deploy a simple "hello world" application that has been instrumented with the Go Language Prometheus library.

Now that your cluster is configured to be able to monitor end-user applications, and your users have permissions to create Monitors and Alerts, you can try monitoring an application to see what it looks like.

1. In the web console, choose the Developer perspective at the top left.
2. Click the Project dropdown and then Create Project
3. Call your project metrical. You can give it a display name or a description if you wish.
4. Click Create

At this point, you can now deploy the sample application. The view you’re immediately presented after creating your Project is the deployment workflow.

5. If for some reason you have navigated away, click +Add on the left first.
6. Click Container Image
7. Where it says Enter an image name, paste the following:

`quay.io/xphyr/prometheus-example-app:v0.5.0`

You can use anything in the box that says Application Name, but you must put prometheus-example-app in the box that says Name.

The Name value is used as the base unique identifier applied to all of the objects that end up deployed. For example, the Deployment will be called metrics-app.

**NOTE** If you are curious to see what is in the demo container image, see the source for the repository located here [https://github.com/xphyr/prometheus-example-app](https://github.com/xphyr/prometheus-example-app)

Leave all other values at their defaults and click Create.

## Creating a Service Monitor

Prometheus doesn’t automatically find application metrics endpoints. It needs to be told where to look. This is done using an instance of a ServiceMonitor.

The following ServiceMonitor definition tells Prometheus to scrape the metrics from the application you just deployed:

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-monitor
spec:
  endpoints:
  - interval: 30s
    port: 8080-tcp
    scheme: http
  selector:
    matchLabels:
      app: prometheus-example-app
```

You’ll notice that the ServiceMonitor is looking at endpoints of a Kubernetes Service, and, in this case, specifically at the port named 8080-tcp. Prometheus will know to find all of the Pods that are a part of this Service and scrape their endpoints. It will do this automatically, no matter how big or small the Deployment is scaled.

1. Copy the above YAML to your clipboard
2. In the Developer perspective, in your metrical Project, click +Add+
3. Click YAML

Paste the ServiceMonitor YAML into the box

4. Click Create

## Viewing Application Metrics

Now that Prometheus is scraping the metrics, you can view the metrics in the OpenShift web console.

1. Make sure you are in the Developer perspective
2. Click Monitoring in the left navigation
3. Click the Metrics tab in the center area
4. Click Select Query and then choose Custom Query

Type http into the box, and notice that a drop-down of options appears

If you recall the Prometheus metrics data from earlier when you visited the /metrics page, you’ll see that these are all metrics that were displayed.

5. Choose http_requests_total and hit Enter
6. Set the graph to 15m(inutes)

You should see a graph of the number of HTTP requests.

## Creating a Custom Alert

Creating custom alerts is just as simple as creating the monitor. Custom alerts are defined using a PrometheusRule object. The following YAML defines a PrometheusRule that will cause an alert to fire when the number of 404 errors in the http_requests_total exceeds a quantity of 10:

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: example-404-alert
spec:
  groups:
  - name: example
    rules:
    - alert: TooManyErrorAlert
      expr: delta(http_requests_total{code="404"}[5m]) > 10
      labels:
        severity: warning
```

1. Make sure you are in the Developer perspective
2. Click +Add
3. Click YAML
4. Copy and paste the above PrometheusRule YAML into the box
5. Click Create

## Test your Alert

We created one simple alert rule, which will fire if there are more than 10 404s in a 5 minute window. To trigger this alert go to the application route and add "/err" to the end of it.  Reload the page many times (15+) This will then trigger an alert within the OpenShift console. To see this alert being created follow these steps:

1. Make sure you are in the Developer perspective
2. Click Monitoring in the left navigation
3. Select the "Alerts" tab and see that the alert is now firing

If your cluster is configured to send alerts per these instructions [Sending notifications to external systems](https://docs.openshift.com/container-platform/4.6/monitoring/managing-alerts.html#sending-notifications-to-external-systems_managing-alerts) user defined alerts can be raised to external systems.

## References

Original Writeup - https://github.com/redhat-scholars/openshift-admins-devops/blob/v4.6blog/documentation/modules/ROOT/pages/metrics-alerting.adoc 