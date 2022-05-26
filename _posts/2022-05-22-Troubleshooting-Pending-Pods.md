---
layout: post
title: "Troubleshooting Kubernetes: Pods in Pending State"
img: k8s-troubleshooting.png
tags: [Kubernetes, Throubleshooting, "101"]
---

## __Introduction__

In my day-to-day work, I help and support teams to shift on containers and cloud technologies, and I could observe that Kubernetes can be overwhelming for newcomers.  
One of the reasons of Kubernetes complexity is because troubleshooting what went wrong can by difficult if you don't know where to look and you need to often look in more than one place.  
I hope this series of blog posts about troubleshooting Kubernetes will help people in their journey.

- [Troubleshooting Kubernetes: ImagePullBackOff]({% post_url 2022-05-21-Troubleshooting-ImagePullBackOff %})
- __Troubleshooting Kubernetes: Pods in Pending State__
- [Troubleshooting Kubernetes: CrashLoopBackOff]({% post_url 2022-05-26-Troubleshooting-CrashLoopBackOff %})

## __Troubleshooting Pods in Pending State__

You deployed your deployment (statefulset or other) and the underlying pod(s) is in `Pending` state.  
There can be various reasons for your pod to be in a pending state, we will go through the process of troubleshooting it with the most two most common errors.

The process it same, find an usefull error message to understand what is going on.
First, let's check the error message you have, for that you will use the `kubectl describe` command.  
This will provide additional information to you on the pod. The output can be long. Let's jump to the `Events` section.

```sh
> kubectl get pod
NAME                                 READY   STATUS    RESTARTS   AGE
whoami-deployment-86d55c94b5-8kgpj   0/1     Pending   0          17s
> kubectl describe pod whoami-deployment-86d55c94b5-8kgpj
```

## __Insufficient CPU__

```sh
> kubectl describe pod whoami-deployment-86d55c94b5-8kgpj
...
...
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  15s   default-scheduler  0/2 nodes are available: 2 Insufficient cpu.
```

Your are interested in the event with `FailedScheduling Reason`.
```sh
0/2 nodes are available: 2 Insufficient cpu.
```
There is 0 out of 2 nodes in the cluster that have enough CPU cores to allocate this pod.  
First of all, let's retrieve the `Request` and `Limits` we have on the deployment by looking at the describe ouput.  
```yaml
Limits:
  cpu:     3
  memory:  128Mi
Requests:
  cpu:     3
  memory:  128Mi
```
We can see we requested 3 CPU core, and we should then ensure the nodes have 3 CPU core.
```sh
> kubectl get node
NAME            STATUS   ROLES           AGE   VERSION
worker-node01   Ready    worker          87m   v1.24.0
worker-node02   Ready    worker          84m   v1.24.0

> kubectl describe node worker-node02
...
...
Capacity:
  cpu:                2
  ephemeral-storage:  32380720Ki
  hugepages-2Mi:      0
  memory:             2035232Ki
  pods:               110
Allocatable:
  cpu:                2
  ephemeral-storage:  29842071503
  hugepages-2Mi:      0
  memory:             1932832Ki
  pods:               110
...
...
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                350m (17%)   0 (0%)
  memory             200Mi (10%)  0 (0%)
...
...
```
So we requested more CPU core than the node has. This would mean that even if we spin more nodes in the cluster Kubernetes will still not be able to schedule it anywhere.

Now let's take a look at the `Allocated resources` part, it shows you all the `Requests` and `Limits` the node has in both unit and percentage.  
Keep in mind that `Request` cannot go over 100% of the node resource but `Limits` can.  
1 CPU core it's either `1` or `1000m`, this means you can ask for half a CPU core by requesting `500m`.

This part tell us that even if we requested 2 or less CPU core, the pod still could be in `Pending` state.  
In our case if we request 1 CPU core, their will be no issue.  
But in the case we request 1800m, Kubernetes will not be able to schedule it anywhere, unless we spin a new node.

## __Insufficient Memory__

```sh
> kubectl describe po whoami-deployment-6d48697874-2hs79
...
...
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  17s   default-scheduler  0/2 nodes are available: 2 Insufficient memory.
```
We will go through the same workflow as above.
We would get our `Request` and `Limits`
```yaml
Limits:
  cpu:     100m
  memory:  2Gi
Requests:
  cpu:     100m
  memory:  2Gi
```
And then retrieve nodes information.
```sh
> kubectl describe node worker-node02
...
...
Capacity:
  cpu:                2
  ephemeral-storage:  32380720Ki
  hugepages-2Mi:      0
  memory:             2035232Ki
  pods:               110
Allocatable:
  cpu:                2
  ephemeral-storage:  29842071503
  hugepages-2Mi:      0
  memory:             1932832Ki
  pods:               110
...
...
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                350m (17%)   0 (0%)
  memory             200Mi (10%)  0 (0%)
...
...
```
The node has `2GB` of total memory and `1932832Ki` free memory, that about `1.9GB`.  
We did not request more memory than the node has but we can see there is already `200Mi` (10%) requested.  
So with this pod request, the total will be over 100% and that is not possible.
We need to either spin a new node for this deployment or lower our `memory request`.

It also possible that the error is a combination of both memory and cpu.
```sh
> kubectl describe po whoami-deployment-6d48697874-2hs79
...
...
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  23s   default-scheduler  0/2 nodes are available: 2 Insufficient cpu, 2 Insufficient memory.
```

## __More Pending State Issues__

The `Pending` state is not necessarily due to CPU or memory, it can be for multiple reasons but you will apply the same workflow:
- Describe the pod
- Find the error message
- Describe other resource pointed out by the error message

I hope this was helpful!