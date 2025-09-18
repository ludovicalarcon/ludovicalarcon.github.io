---
layout: post
title: PriorityClasses are underrated but critical
img: k8s-logo.png
tags: [kubernetes]
---

When people think about scheduling, they usually go to affinity rules, topology constraints, taints, etc...  
Yet there is another mechanism that is taken into account by the scheduler but almost always gets left out.  
But `PriorityClasses` are critical, that will decide which pod die and which live when the cluster is under pressure.  

Most teams either ignore them or are not aware of them and that's too bad.  
Used properly, `PriorityClasses` ensure critical workloads continue running even under resource constraints.  
Basically, it allows to assign different priorities to pods that will then be used by the scheduler for scheduling but also eviction.  
Higher priority pods are scheduled first and lower ones are evicted first in case of pressure.  

### PriorityClass

A `PriorityClass` is a cluster-wide resource that assigns a different priority to pods.  
It will then be used by the scheduler to make decision for scheduling but also eviction.  
Higher priority pods are scheduled first and lower ones are evicted first in case of pressure.  

Simple example for definition:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: system-critical
value: 1000000
globalDefault: false
description: "Critical system Pods."
```

And how to attach it:

```yaml
...
...
spec:
  template:
    spec:
      priorityClassName: system-critical
...
...
```

These pods will now be scheduled before anything else and not be evicted under node pressure.  

### Default situation

So now let's see what happens when no priorityClasses are used.  
When no priorityClass is set, pods get assigned a default priority with a value of 0.  
So concretely, if none of the workloads are using priorityClasses they are all treated equally.  
Your batch jobs will have the same importance as your critical workloads. If your cluster is in bad shape  
and pods need to be evicted, it will become Russian roulette eviction.  
This situation is a hidden risk, hence adding a simple line on critical pods can save you from major incident.  

### Priority strategy

It's time to think about priority strategy because defining everything as `critical` leads to nothing is.  
I personally think that 4 tiers are quite a good strategy

```yaml
# For DNS, CNI, admission controllers, etc.
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: system-critical
value: 1000000

# For critical pods
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: pod-critical
value: 90000

# For production pods
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: prod
value: 10000


# For batch
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch
value: 1000
```

So with that, the scheduler will have knowledge of what is important in your cluster.  
In case of resource pressure, it will be able to:

- kill low priority pods without touching critical ones
- make room for critical pods
- reschedule low priority pods elsewhere

### Audit current state

Below command will show the deployed `PriorityClasses` objects

```sh
kubectl get pc
```

And this one liner will list and sort all deployed pods by PriorityClasses

```sh
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.priorityClassName}{"\n"}{end}' | sort
```

### Conclusion

`PriorityClasses` are in my opinion one of the simplest yet underrated scheduling feature in Kubernetes.  
It really can prevent incidents but also avoid to worsening an ongoing incident by evicting critical workloads.  
I hope this post was useful and makes you use PriorityClasses if that wasn't the case.  
