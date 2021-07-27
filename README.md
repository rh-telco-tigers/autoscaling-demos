# Autoscaling Pods and Machines in OpenShift

## Table of Contents

<!-- TOC -->
- [Autoscaling Pods and Machines in OpenShift](#autoscaling-pods-and-machines-in-openshift)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Prerequisites](#prerequisites)
  - [Horizontal Pod Autoscaling](#horizontal-pod-autoscaling)
    - [Deploying a sample app that will use memory](#deploying-a-sample-app-that-will-use-memory)
    - [Create Horizontal Pod Autoscaler](#create-horizontal-pod-autoscaler)
    - [Exercising the Horizontal Pod Autoscaler](#exercising-the-horizontal-pod-autoscaler)
  - [Vertical Pod Autoscaling](#vertical-pod-autoscaling)
    - [Installing VPA Operator](#installing-vpa-operator)
    - [Implementing the VPA](#implementing-the-vpa)
  - [Cluster Autoscaling](#cluster-autoscaling)
    - [Create a Cluster Autoscaler](#create-a-cluster-autoscaler)
    - [Create a Machine Autoscaler](#create-a-machine-autoscaler)
    - [Exercising the Autoscaling Feature](#exercising-the-autoscaling-feature)
    - [Cleanup AutoScaling](#cleanup-autoscaling)
  - [Advanced Topics](#advanced-topics)
    - [Using Spot instances in AWS with AutoScaling](#using-spot-instances-in-aws-with-autoscaling)
    - [Create a new machineSet for spot instances](#create-a-new-machineset-for-spot-instances)
    - [AutoScaler - Bonus Round](#autoscaler---bonus-round)
    - [Cleanup Spot AutoScaler](#cleanup-spot-autoscaler)
<!-- TOC -->

## Introduction

This guide will take you through leveraging the OpenShift autoscaling functionality. This includes both the Horizontal Pod Autoscaler, Vertical Pod Autoscaler and finally the  Machine Autoscaler. The steps provided below will work for any OpenShift cluster that was built with the OpenShift "IPI" installer process (such as AWS, Azure and VMWare), however the [Advanced Topics](#advanced-topics) section will focus in on the use of AWS and "spot instances".

## Prerequisites

In order to exercise the Machine Autoscaler, you will need to have a cluster built with the IPI installer process. Please see the [OpenShift Install Documentation](https://docs.openshift.com/container-platform/4.7/welcome/index.html) for the process to install OpenShift.

## Horizontal Pod Autoscaling

This guide will take you through leveraging the OpenShift Machine Autoscaler in an AWS hosted cluster. The steps provided below will work for any OpenShift cluster that was built with the OpenShift "IPI" installer process (such as AWS, Azure and VMWare).

### Deploying a sample app that will use memory

Start by creating a new project `oc new-project memuser`

We will use a small program called [memuser](https://gitlab.com/xphyr/k8s_memuser) that can allocate and hold onto memory, or de-allocate memory based on two http endpoints. You are welcome to compile your own version of the program or, you may use a prebuilt image hosted on Quay.io [here](https://quay.io/repository/xphyr/memuser). The application definitions in this repo will automatically use the Quay.io hosted image.

Start by deploying the memuser application and creating a service and route within OpenShift:

```shell
$ oc new-project memuser
$ oc create -f podAutoscaling/deployment.yml
deployment.apps/memuser created
$ oc create -f podAutoscaling/service.yml
service/memuserservice created
$ oc expose svc/memuserservice
$ oc get route
NAME             HOST/PORT                                       PATH   SERVICES         PORT       TERMINATION   WILDCARD
memuserservice   memuserservice-memuser.apps.mark.aws3.ocp.run         memuserservice   8080-tcp                 None
```

We have now deployed our memuser application and made it available to the outside world.  Lets curl the endpoint and see what we get. Be sure to update the URL in the commands below to point to your specific url.

```shell
$ curl http://memuserservice-memuser.apps.mark.aws3.ocp.run/consumemem
Hello User. My current memory usage is:
 Alloc = 50 MiB	 Sys = 70 MiB	 NumGC = 4
# Lets run it a few more times
$ curl http://memuserservice-memuser.apps.mark.aws3.ocp.run/consumemem
Hello User. My current memory usage is:
 Alloc = 100 MiB  Sys = 136 MiB	 NumGC = 5
$ curl http://memuserservice-memuser.apps.mark.aws3.ocp.run/consumemem
Hello User. My current memory usage is:
 Alloc = 150 MiB	Sys = 136 MiB	 NumGC = 5
```

Note that each time you run it, it consumes 50Mb more. Now let's run a GC and clean up the memory currently in use:

```shell
$ curl http://memuserservice-memuser.apps.mark.aws3.ocp.run/clearmem
Memory has been cleared.
 Alloc = 0 MiB	Sys = 202 MiB	 NumGC = 8
```

Note that the Alloc memory is now 0 MiB but Sys is 202 MiB. This is due to the way Go handles memory allocation. It will eventually give that memory up. So we now have a way to consume memory, and to clear up that memory usage. The program is designed to only use so much memory though, to ensure we don't crash your server. Lets try running the memory usage up. Copy/paste the following script and run it:

```shell
$ for i in {1..30}; do curl http://memuserservice-hpademo.apps.mark.aws3.ocp.run/consumemem; done
$ curl http://memuserservice-memuser.apps.mark.aws3.ocp.run/clearmem
```

Note that eventually Alloc stops growing. This is a failsafe to ensure you don't run out of memory.

### Create Horizontal Pod Autoscaler

Now that we have a way to consume and clean up memory, lets create a horizontal pod autoscaler that scales based on memory. Review the file "hpa-memuser.yml" and then apply to your cluster:

```shell
# First lets clear the memory usage
$ oc create -f podAutoscaling/hpa-memuser.yml
$ oc get hpa
```

This autoscaler will scale when the memory usage of a given pod goes above 80%. It will then try to keep the average memory usage across all the deployed pods at or below 80%. Lets watch the autoscaler work.

### Exercising the Horizontal Pod Autoscaler

```shell
$ watch oc get hpa
```

Now in a new window, run our curl command again to start ramping up our memory usage:

```shell
$ for i in {1..30}; do curl http://memuserservice-memuser.apps.mark.aws3.ocp.run/consumemem; done
```

Now go back and check the other window. You should start to see the hpa scale up your workload based on the overall memory usage. It may take a minute or two for it to pick up the usage and start to scale up. Re-run the for loop again, and watch the output of the hpa. It will continue to scale up to a max of 10 pods.

HorizontalPodAutoscaler will also scale pods back down when not using the memory. Lets clear the memory usage so we can see the pods scale back down. Run `http://memuserservice-memuser.apps.mark.aws3.ocp.run/clearmem` until all pods are showing no memory allocations. Wait about 5 minutes, you should see the horizontal pod autoscaler start to scale the number of pods back down again. If you want to see it slowly scale back, only run the clearmem command 2 times to ensure you leave some pods still using memory.

For additional information on the use of Horizontal Pod Autoscalers, see [https://docs.openshift.com/container-platform/4.7/nodes/pods/nodes-pods-autoscaling.html](https://docs.openshift.com/container-platform/4.7/nodes/pods/nodes-pods-autoscaling.html)

## Vertical Pod Autoscaling

The OpenShift Container Platform Vertical Pod Autoscaler Operator (VPA) automatically reviews the historic and current CPU and memory resources for containers in pods and can update the resource limits and requests based on the usage values it learns. The Vertical Pod Autoscaler (VPA) is not installed by default within your cluster, however it can be easily enabled through the use of OperatorHub.

> **NOTE:** VPA is a Technology Preview feature only as of OpenShift 4.7. Technology Preview features are not supported with Red Hat production service level agreements (SLAs) and might not be functionally complete.

### Installing VPA Operator

1.  Log into your OpenShift Console
2.  Select Operators->OperatorHub
3.  Search for "Vertical Pod"
4.  Select the "Vertical Pod Autoscaler and click "Install"
5.  Accept all defaults and click Install

### Implementing the VPA

We will start by creating an instance of a VerticalPodAutoscaler in our "memuser" project. We will be deploying our VPA in the "Recommend" mode, where the VPA will take no action on your deployment, but will calculate recommendations on where your requests and limits should be set.

```shell
$ oc project memuser
$ oc get pods
NAME                       READY   STATUS    RESTARTS   AGE
memuser-66686fc697-4grm6   1/1     Running   0          112s
$ oc apply -f podAutoscaling/vpa-memuser.yml
verticalpodautoscaler.autoscaling.k8s.io/vpa-recommender created
```

If you open the vpa-memuser.yml file you will see that "updateMode" is set to "off". Other options are:

* **Auto** or **Recreate** - VPA deletes existing pods in the project that are out of alignment with its recommendations
* **Inital** - applies VPA recommendation at pod creation time
* **Off** - only provides recommended resource limits and requests, but does not update your deployments

The VPA will take a few minutes to build its first recomendation. 

```shell
$ oc get vpa vpa-recommender --output yaml
...
status:
  conditions:
  - lastTransitionTime: "2021-07-20T19:12:43Z"
    status: "True"
    type: RecommendationProvided
  recommendation:
    containerRecommendations:
    - containerName: memuser
      lowerBound:
        cpu: 25m
        memory: 262144k
      target:
        cpu: 25m
        memory: 262144k
      uncappedTarget:
        cpu: 25m
        memory: 262144k
      upperBound:
        cpu: 1595m
        memory: "56957093523"
```

> **NOTE:** The output above is truncated, your output will be much longer. Focus in on the "status" section of the output to view the important information. 

In the above output you will see that the recommendation for this particular deployment is between 25 milicores and 1.5 milicores. Memory recommendations will take longer to stabilize.

For additional information on the use of Vertical Pod Autoscalers, see [https://docs.openshift.com/container-platform/4.7/nodes/pods/nodes-pods-vertical-autoscaler.html](https://docs.openshift.com/container-platform/4.7/nodes/pods/nodes-pods-vertical-autoscaler.html)

## Cluster Autoscaling

### Create a Cluster Autoscaler

The cluster autoscaler adds and removes machines from a cluster to match the workload demand. The Cluster Autoscaler is managed by an operator. Cluster Autoscaling is only supported on clusters where the machine API is operational:

* AWS
* Azure
* vSphere
* Google Cloud
* Bare Metal

To view the operator, execute the following:

`oc get deployments -n openshift-machine-api cluster-autoscaler-operator`

Create an instance of the cluster-autoscaler-operator

`oc create -f machineAutoscaling/clusterAutoScaler.yml`

Validate that the clusterAutoScaler was created

`oc get clusterautoscaler -n openshift-machine-api`

See the pods running that make this possible

`oc get pod -n openshift-machine-api`

### Create a Machine Autoscaler

Now that we have a cluster autoscaler in place, we need to create a machine autoscaling policy.

Start by listing out the machineSets that you have available in your cluster:

`oc get machinesets -n openshift-machine-api`

We will leverage these existing machineSets in our MachineAutoScaler definition. Update the `machineAutoscaling/machineAutoScaler.yml` file, replacing \<machineSetName\> with the name of a machine set from the above output. If you want to enable autoscaling for mulitple machine sets, create multiple copies of the `machineAutoscaling/machineAutoScaler.yml` for each machineset you wish to autoexpand updating the required fields as appropriate.

Apply the machineAutoScaler files to your cluster:

```shell
$ oc create -f machineAutoscaling/machineAutoScaler.yml
$ oc get machineautoscaler -n openshift-machine-api
NAME                                     REF KIND     REF NAME                       MIN   MAX   AGE
autoscale-mark-vwwcf-worker-us-east-2a   MachineSet   mark-vwwcf-worker-us-east-2a   1     3     6h2m
autoscale-mark-vwwcf-worker-us-east-2b   MachineSet   mark-vwwcf-worker-us-east-2b   1     3     6h2m
```

### Exercising the Autoscaling Feature

In order to test the autoscaling we will use a workqueue. Our workqueue will build up a backlog of pods that need to be run. The pods start up a busybox instances and then pause for 300 seconds.

```shell
$ 
$ oc create -f machineAutoscaling/work-queue.yml
$ oc get jobs
```

Check to see that you have pods pending by running `oc get pods`. You should see a number of pods in a Pending state. Check the state of your machineAutoScaler:

```
$ oc get machineautoscaler -n openshift-machine-api
NAME                           REF KIND     REF NAME             MIN   MAX   AGE
autoscale-ocp47-jzgl9-worker   MachineSet   ocp47-jzgl9-worker   4     6     2m23s
```


### Cleanup AutoScaling

If you want to remove the ability for your cluster to autoscale, you have two options. The first is to edit the machineAutoScaler instances and set the "min" and "max to the same value. The second (which we will do here) is to delete the machineAutoScaler instances.

```shell
$ oc get machineautoscaler -n openshift-machine-api
NAME                                     REF KIND     REF NAME                       MIN   MAX   AGE
autoscale-mark-vwwcf-worker-us-east-2a   MachineSet   mark-vwwcf-worker-us-east-2a   1     3     6h2m
autoscale-mark-vwwcf-worker-us-east-2b   MachineSet   mark-vwwcf-worker-us-east-2b   1     3     6h2m
$ oc delete machineautoscaler/autoscale-mark-vwwcf-worker-us-east-2a -n openshift-machine-api
$ oc delete machineautoscaler/autoscale-mark-vwwcf-worker-us-east-2b -n openshift-machine-api
```

## Advanced Topics

### Using Spot instances in AWS with AutoScaling

AWS has the concept of [spot instances](https://aws.amazon.com/aws-cost-management/aws-cost-optimization/spot-instances/). These can be a less costly way to run your cluster and handle burst requirements at a low cost, with some limitations. Be sure to fully understand how spot instances work before implementing them in your cluster. This example will show you how to create a machineset that leverages spot instances, and scales them up based on demand, and scales them down again when they are no longer needed.

### Create a new machineSet for spot instances

Creation of a spot instance machine leverages information from your existing machineSets. We will need to do some editing of this file before re-applying it to your cluster. To start, get a list of your machineSets:

```shell
$ oc get machinesets -n openshift-machine-api
NAME                           DESIRED   CURRENT   READY   AVAILABLE   AGE
mark-vwwcf-worker-us-east-2a   1         1         1       1           26h
mark-vwwcf-worker-us-east-2b   1         1         1       1           26h
mark-vwwcf-worker-us-east-2c   1         1         1       1           26h
```

In the above output you can see we have three machinesets. One per region. In this example, we will create two new machinesets, one in 2a, and one in 2b, and they will be configured to create a spot instance. We will also set them to have 0 machines to start.  To begin we will get the yaml definition for each of these existing machinesets:

```shell
$ oc get machineset/mark-vwwcf-worker-us-east-2a -n openshift-machine-api -o yaml > spot-worker-us-east2a.yaml
$ oc get machineset/mark-vwwcf-worker-us-east-2b -n openshift-machine-api -o yaml > spot-worker-us-east2b.yaml
```

Edit the yaml removing the following sections:

* metadata.managedfields.*
* metadata.generation
* status.*
* metadata.resourceVersion
* metadata.selfLink
* metadata.uid
* metadata.creationTimestamp
* metadata.annotations.autoscaling

Update all refrences to your machineset name to indicate they are spot instances. For example `mark-vwwcf-worker-us-east-2a` becomes `mark-vwwcf-worker-us-east-2a-spot`. Add two entries to spec.template.spec. The first will be an additional node label `spec.template.spec.metadata.labels.spotInstance: "true"` and the second is `spec.template.spec.providerSpec.valuespotMarketOptions: {}` so it looks like the following code snipet:

```yaml
  spec:
   metadata: 
    labels:
        spotInstance: "true"
   providerSpec:
     value:
       spotMarketOptions: {}
       userDataSecret:
```

Finally update `spec.replicas: 0` so we dont spin any of these machines up.

Save this file and repeat for the additional zones you wish to use spot instances from.

Apply the new machineSets to your cluster using the oc command:

```shell
$ oc create -f spot-worker-us-east2a.yaml
$ oc create -f spot-worker-us-east2b.yaml
$ oc get machinesets -n openshift-machine-api
NAME                                DESIRED   CURRENT   READY   AVAILABLE   AGE
mark-vwwcf-worker-us-east-2a        1         1         1       1           27h
mark-vwwcf-worker-us-east-2a-spot   0         0                             16s
mark-vwwcf-worker-us-east-2b        1         1         1       1           27h
mark-vwwcf-worker-us-east-2b-spot   0         0                             10s
mark-vwwcf-worker-us-east-2c        1         1         1       1           27h
```

Note that the spot machine instances now exist, but there are no machines currently deployed. Using the instructions above for [Create a Machine Autoscaler](#create-a-machine-autoscaler) we will create one or more machineautoscalers that use the new spot instance machinesets that you created. Update the `minReplicas` to be 0 so that we are only running these spot instances when they are required.

Apply this yaml and validate that the machineautoscalers are in place:

```shell
$ oc create -f demo-us-east-2a-spot.yml
$ oc create -f demo-us-east-2b-spot.yml
$ oc get machineautoscaler -n openshift-machine-api
NAME                                          REF KIND     REF NAME                            MIN   MAX   AGE
autoscale-mark-vwwcf-worker-us-east-2a-spot   MachineSet   mark-vwwcf-worker-us-east-2a-spot   0     3     13s
autoscale-mark-vwwcf-worker-us-east-2b-spot   MachineSet   mark-vwwcf-worker-us-east-2b-spot   0     3     9s
$ oc get machinesets -n openshift-machine-api
NAME                                DESIRED   CURRENT   READY   AVAILABLE   AGE
mark-vwwcf-worker-us-east-2a        1         1         1       1           27h
mark-vwwcf-worker-us-east-2a-spot   0         0                             16s
mark-vwwcf-worker-us-east-2b        1         1         1       1           27h
mark-vwwcf-worker-us-east-2b-spot   0         0                             10s
mark-vwwcf-worker-us-east-2c        1         1         1       1           27h
```

Note that the autoscaler exists, but has not started any spot instances. We will use a job queue that is designed to target the spot instances to exercise this autoscaler. Look at the file `machineAutoscaling/work-queue-spot.yml`.  Note that there is a field "nodeSelector.spotInstance: true". This tells kubernetes to only schedule this on nodes that have that label. If you remember when we created the new "spot" machineSets we added a label called "spotInstance: true".

```shell
$ oc new-project work-queue
$ oc create -f machineAutoscaling/work-queue-spot.yml
$ oc get jobs -w
# when you are done watching the jobs hit CTRL+C to stop the watch command
```

Open another terminal window, and run `watch oc get machinesets -n openshift-machine-api` you will see the number of nodes in your cluster scale up, and then as the job completes they will scale back down to zero.

### AutoScaler - Bonus Round

Let's say the jobs are not running fast enough for you, and you want to allow the autoScaler to go larger, how do you do this? Lets edit the autoScalers that we have defined, and let it go up to 6.

```shell
$ oc get machineautoscaler -n openshift-machine-api
NAME                                          REF KIND     REF NAME                            MIN   MAX   AGE
autoscale-mark-vwwcf-worker-us-east-2a-spot   MachineSet   mark-vwwcf-worker-us-east-2a-spot   0     3     13s
autoscale-mark-vwwcf-worker-us-east-2b-spot   MachineSet   mark-vwwcf-worker-us-east-2b-spot   0     3     9s
# edit the max value for each autoscaler
$ oc edit machineautoscaler/autoscale-mark-vwwcf-worker-us-east-2a-spot -n openshift-machine-api
$ oc edit machineautoscaler/autoscale-mark-vwwcf-worker-us-east-2a-spot -n openshift-machine-api
```

Now just sit back and watch ... nothing happens!  Wait, what is going on here, I told the autoscaler to scale this up by at least two more nodes per zone. You did, but we have a failsafe in place to keep the cluster from scaling out of control.

```shell
$ oc describe clusterautoscaler -n openshift-machine-api
Name:         default
Namespace:
Labels:       <none>
...
Spec:
  Pod Priority Threshold:  -10
  Resource Limits:
    Max Nodes Total:  12
```

Note that last line "Max Nodes Total: 12" this is the total nodes for your entire cluster (including Control Plane and Worker nodes). How many nodes are in your cluster?

```shell
$ oc get nodes --no-headers | wc -l
12
```

Since we configured the ClusterAutoScaler to never allow more than 12 nodes, no matter how you configure the MachineAutoScaler, it will not allow the cluster to grow over 12. If you want to allow your cluster to grow larger, run the following command and edit the maxNodes value.

`$ oc edit clusterautoscaler/default -n openshift-machine-api`

Check the state of your machineSets now and see that OpenShift is now scaling up to the max new number of nodes you requested.

```shell
$ oc get machinesets -n openshift-machine-api
NAME                                DESIRED   CURRENT   READY   AVAILABLE   AGE
mark-vwwcf-worker-us-east-2a        1         1         1       1           28h
mark-vwwcf-worker-us-east-2a-spot   4         4         3       4           74m
mark-vwwcf-worker-us-east-2b        1         1         1       1           28h
mark-vwwcf-worker-us-east-2b-spot   4         4         3       4           74m
mark-vwwcf-worker-us-east-2c        1         1         1       1           28h
```

### Cleanup Spot AutoScaler

```shell
$ oc get machineautoscaler -n openshift-machine-api
NAME                                          REF KIND     REF NAME                            MIN   MAX   AGE
autoscale-mark-vwwcf-worker-us-east-2a-spot   MachineSet   mark-vwwcf-worker-us-east-2a-spot   0     3     14m
autoscale-mark-vwwcf-worker-us-east-2b-spot   MachineSet   mark-vwwcf-worker-us-east-2b-spot   0     3     14m
$ oc delete machineautoscaler/autoscale-mark-vwwcf-worker-us-east-2a-spot -n openshift-machine-api
$ oc delete machineautoscaler/autoscale-mark-vwwcf-worker-us-east-2b-spot -n openshift-machine-api
$ oc get machinesets -n openshift-machine-api
NAME                                DESIRED   CURRENT   READY   AVAILABLE   AGE
mark-vwwcf-worker-us-east-2a        1         1         1       1           27h
mark-vwwcf-worker-us-east-2a-spot   0         0                             16s
mark-vwwcf-worker-us-east-2b        1         1         1       1           27h
mark-vwwcf-worker-us-east-2b-spot   0         0                             10s
mark-vwwcf-worker-us-east-2c        1         1         1       1           27h
$ oc delete machineset/mark-vwwcf-worker-us-east-2a-spot -n openshift-machine-api
$ oc delete machineset/mark-vwwcf-worker-us-east-2b-spot -n openshift-machine-api
```