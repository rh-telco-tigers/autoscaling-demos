# Using Cluster Autoscaler

## Table of Contents

<!-- TOC -->
- [Using Cluster Autoscaler](#using-cluster-autoscaler)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Prerequisites](#prerequisites)
  - [Leveraging Cluster Autoscaling](#leveraging-cluster-autoscaling)
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

This guide will take you through leveraging the OpenShift Machine Autoscaler in an AWS hosted cluster. The steps provided below will work for any OpenShift cluster that was built with the OpenShift "IPI" installer process (such as AWS, Azure and VMWare).

## Prerequisites

You will need to have a cluster built with the IPI installer process. Please see the [OpenShift Install Documentation](https://docs.openshift.com/container-platform/4.6/welcome/index.html) for the process to install OpenShift.

## Leveraging Cluster Autoscaling

### Create a Cluster Autoscaler

The cluster autoscaler adds and removes machines from a cluster to match the workload demand. The Cluster Autoscaler is
managed by an operator.

To view the operator, execute the following:

`oc get deployments -n openshift-machine-api cluster-autoscaler-operator`

Create an instance of the cluster-autoscaler-operator

`oc create -f templates/clusterAutoScaler.yml`

Validate that the clusterAutoScaler was created

`oc get clusterautoscaler -n openshift-machine-api`

See the pods running that make this possible

`oc get pod -n openshift-machine-api`

### Create a Machine Autoscaler

Now that we have a cluster autoscaler in place, we need to create a machine autoscaling policy.

Start by listing out the machineSets that you have available in your cluster:

`oc get machinesets -n openshift-machine-api`

We will leverage these existing machineSets in our MachineAutoScaler definition. Update the `templates/machineAutoScaler.yml` file, replacing \<machineSetName\> with the name of a machine set from the above output.  If you want to enable autoscaling for mulitple machine sets, create multiple copies of the `templates/machineAutoScaler.yml` for each machineset you wish to autoexpand updating the required fields as appropriate.

Apply the machineAutoScaler files to your cluster:

```
$ oc create -f templates/machineAutoScaler.yml
$ oc get machineautoscaler -n openshift-machine-api
NAME                                     REF KIND     REF NAME                       MIN   MAX   AGE
autoscale-mark-vwwcf-worker-us-east-2a   MachineSet   mark-vwwcf-worker-us-east-2a   1     3     6h2m
autoscale-mark-vwwcf-worker-us-east-2b   MachineSet   mark-vwwcf-worker-us-east-2b   1     3     6h2m
```

### Exercising the Autoscaling Feature

In order to test the autoscaling we will use a workqueue 

```
$ oc new-project work-queue
$ oc create -f templates/work-queue.yml
$ oc get jobs
```

### Cleanup AutoScaling

If you want to remove the ability for your cluster to autoscale, you have two options. The first is to edit the machineAutoScaler instances and set the "min" and "max to the same value. The second (which we will do here) is to delete the machineAutoScaler instances.

```
$ oc get machineautoscaler -n openshift-machine-api
NAME                                     REF KIND     REF NAME                       MIN   MAX   AGE
autoscale-mark-vwwcf-worker-us-east-2a   MachineSet   mark-vwwcf-worker-us-east-2a   1     3     6h2m
autoscale-mark-vwwcf-worker-us-east-2b   MachineSet   mark-vwwcf-worker-us-east-2b   1     3     6h2m
$ oc delete machineautoscaler/autoscale-mark-vwwcf-worker-us-east-2a -n openshift-machine-api
$ oc delete machineautoscaler/autoscale-mark-vwwcf-worker-us-east-2b -n openshift-machine-api
```

## Advanced Topics

### Using Spot instances in AWS with AutoScaling

AWS has the concept of [spot instances](https://aws.amazon.com/aws-cost-management/aws-cost-optimization/spot-instances/). These can be a less costly way to run your cluster and handle burst requirements at a low cost, with some limitations. Be sure to fully understand how spot instances work before implementing them in your cluster. This example will show you how to create a machineset that leverages spot instances, and scales them up based on demand, and 

### Create a new machineSet for spot instances

Creation of a spot instance machine leverages information from your existing machineSets. We will need to do some editing of this file before re-applying it to your cluster. To start, get a list of your machineSets:

```
$ oc get machinesets -n openshift-machine-api
NAME                           DESIRED   CURRENT   READY   AVAILABLE   AGE
mark-vwwcf-worker-us-east-2a   1         1         1       1           26h
mark-vwwcf-worker-us-east-2b   1         1         1       1           26h
mark-vwwcf-worker-us-east-2c   1         1         1       1           26h
```

In the above output you can see we have three machinesets. One per region. In this example, we will create two new machinesets, one in 2a, and one in 2b, and they will be configured to create a spot instance. We will also set them to have 0 machines to start.  To begin we will get the yaml definition for each of these existing machinesets:

```
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

```
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

``` 
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

```
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

Note that the autoscaler exists, but has not started any spot instances. We will use a job queue that is designed to target the spot instances to exercise this autoscaler. Look at the file `templates/work-queue-spot.yml`.  Note that there is a field "nodeSelector.spotInstance: true". This tells kubernetes to only schedule this on nodes that have that label. If you remember when we created the new "spot" machineSets we added a label called "spotInstance: true".

```
$ oc new-project work-queue
$ oc create -f templates/work-queue-spot.yml
$ oc get jobs -w
# when you are done watching the jobs hit CTRL+C to stop the watch command
```

Open another terminal window, and run `watch oc get machinesets -n openshift-machine-api` you will see the number of nodes in your cluster scale up, and then as the job completes they will scale back down to zero.

### AutoScaler - Bonus Round

Let's say the jobs are not running fast enough for you, and you want to allow the autoScaler to go larger, how do you do this? Lets edit the autoScalers that we have defined, and let it go up to 6.

```
$ oc get machineautoscaler -n openshift-machine-api
NAME                                          REF KIND     REF NAME                            MIN   MAX   AGE
autoscale-mark-vwwcf-worker-us-east-2a-spot   MachineSet   mark-vwwcf-worker-us-east-2a-spot   0     3     13s
autoscale-mark-vwwcf-worker-us-east-2b-spot   MachineSet   mark-vwwcf-worker-us-east-2b-spot   0     3     9s
# edit the max value for each autoscaler
$ oc edit machineautoscaler/autoscale-mark-vwwcf-worker-us-east-2a-spot -n openshift-machine-api
$ oc edit machineautoscaler/autoscale-mark-vwwcf-worker-us-east-2a-spot -n openshift-machine-api
```

Now just sit back and watch ... nothing happens!  Wait, what is going on here, I told the autoscaler to scale this up by at least two more nodes per zone. You did, but we have a failsafe in place to keep the cluster from scaling out of control.

```
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

```
$ oc get nodes --no-headers | wc -l
12
```

Since we configured the ClusterAutoScaler to never allow more than 12 nodes, no matter how you configure the MachineAutoScaler, it will not allow the cluster to grow over 12. If you want to allow your cluster to grow larger, run the following command and edit the maxNodes value.

`$ oc edit clusterautoscaler/default -n openshift-machine-api`

Check the state of your machineSets now and see that OpenShift is now scaling up to the max new number of nodes you requested.

```
$ oc get machinesets -n openshift-machine-api
NAME                                DESIRED   CURRENT   READY   AVAILABLE   AGE
mark-vwwcf-worker-us-east-2a        1         1         1       1           28h
mark-vwwcf-worker-us-east-2a-spot   4         4         3       4           74m
mark-vwwcf-worker-us-east-2b        1         1         1       1           28h
mark-vwwcf-worker-us-east-2b-spot   4         4         3       4           74m
mark-vwwcf-worker-us-east-2c        1         1         1       1           28h
```

### Cleanup Spot AutoScaler

```
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