# Autoscaling an application based on Memory Usage

## Table of Contents

<!-- TOC -->
- [Autoscaling an application based on Memory Usage](#autoscaling-an-application-based-on-memory-usage)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Creating a sample app that will use memory](#creating-a-sample-app-that-will-use-memory)
<!-- TOC -->

## Introduction

This guide will take you through leveraging the OpenShift Machine Autoscaler in an AWS hosted cluster. The steps provided below will work for any OpenShift cluster that was built with the OpenShift "IPI" installer process (such as AWS, Azure and VMWare).

## Creating a sample app that will use memory

Start by creating a new project `oc new-project memuser`

We will use a small program called [memuser](https://gitlab.com/xphyr/k8s_memuser) that can allocate and hold onto memory, or deallocate memory based on two http endpoints. You are welcome to compile your own version of the program or, you may use a prebuilt image hosted on Quay.io [here](https://quay.io/repository/xphyr/memuser). The application definitions in this repo will automatically use the Quay.io hosted image. 

Start by deploying the memuser application and creating a service and route within OpenShift:

```
$ oc new-project memuser
$ oc create -f deployment.yml
deployment.apps/memuser created
$ oc create -f service.yml
service/memuserservice created
$ oc expose svc/memuserservice
$ oc get route
NAME             HOST/PORT                                       PATH   SERVICES         PORT       TERMINATION   WILDCARD
memuserservice   memuserservice-memuser.apps.cluster.example.com         memuserservice   8080-tcp                 None
```

We have now deployed our memuser application and made it available to the outside world.  Lets curl the endpoint and see what we get:

```
$ curl http://memuserservice-memuser.apps.mark.aws3.ocp.run
Hello User. My current memory usage is:
 Alloc = 50 MiB	 TotalAlloc = 50 MiB	 Sys = 70 MiB	 NumGC = 4
# Lets run it a few more times
$ curl http://memuserservice-memuser.apps.mark.aws3.ocp.run
Hello User. My current memory usage is:
 Alloc = 100 MiB	 TotalAlloc = 100 MiB	 Sys = 136 MiB	 NumGC = 5
$ curl http://memuserservice-memuser.apps.mark.aws3.ocp.run
Hello User. My current memory usage is:
 Alloc = 150 MiB	 TotalAlloc = 100 MiB	 Sys = 136 MiB	 NumGC = 5
```

Note that each time you run it, it consumes 50Mb more. Now let's run a GC and clean up the memory currently in use:

```
$ curl http://memuserservice-memuser.apps.mark.aws3.ocp.run/clearmem
Memory has been cleared.
 Alloc = 0 MiB	 TotalAlloc = 150 MiB	 Sys = 202 MiB	 NumGC = 8
```

Note that the Alloc memory is now 0 MiB but Sys is 202 MiB. This is due to the way Go handles memory allocation. It will eventually give that memory up. For now, lknow that we have reduced our memory usage. So we now have a way to consume memory, and to clear up that memory usage. The program is designed to only use so much memory though, to ensure we dont crash your server.  Lets try running the memory usage up.  Copy/paste the following script and run it:

```
$ for i in {1..30}; do curl http://memuserservice-memuser.apps.mark.aws3.ocp.run/; done
$ curl http://memuserservice-memuser.apps.mark.aws3.ocp.run/clearmem
```

Note that eventually Alloc stops growing. This is a failsafe to ensure you dont run out of memory.

Now that we have a way to consume and clean up memory, lets create a horizontal pod autoscaler that scales based on memory. Review the file "hpa-memuser.yml" and then apply to your cluster:

```
# First lets clear the memory usage
$ oc create -f hpa-memuser.yml
```

This autoscaler will scale when the memory usage of a given pod goes above 80%. It will then try to keep the average memory usage across all the deployed pods at or below 80%. Lets watch the autoscaler work.

```
$ watch oc get hpa
```

Now in a new window, run our curl command again to start ramping up our memory usage:

```
$ for i in {1..30}; do curl http://memuserservice-memuser.apps.mark.aws3.ocp.run/; done
```

Now go back and check the other window. You should start to see the hpa scale up your workload based on the overall memory usage. It may take a minute or two for it to pick up the usage and start to scale up.  Re-run the for loop again, and watch the output of the hpa. It will continue to scale up to a max of 10 pods.

HorizontalPodAutoscaler will also scale pods back down when not using the memory. Lets clear the memory usage so we can see the pods scale back down. Run `http://memuserservice-memuser.apps.mark.aws3.ocp.run/clearmem` until all pods are showing no memory allocations. Wait about 5 minutes, you should see the horizontal pod autoscaler start to scale the number of pods back down again. If you want to see it slowly scale back, only run the clearmem command 2 times to ensure you leave some pods still using memory.
