---
layout: post
title: Prevent chaos with Resources Quota
img: k8s-logo.png
tags: [kubernetes]
---

With the increasing usage of multi tenancy Kubernetes clusters, a problematic rise: “How to properly share resources and avoid one team hogging all?”  

`ResourceQuotas` exits to answer this.  
Yet, I noticed that most people either ignore them or don’t know about it.  

## What is a ResourceQuota?

From documentation:
> A ResourceQuota provides constraints that limit aggregate resource consumption per namespace.
> A ResourceQuota can also limit the quantity of objects that can be created in a namespace by API kind, as well as the total amount of infrastructure resources that may be consumed by API objects found in that namespace.

So in a nutshell, a `ResourceQuota` is a Kubernetes object that restricts resource consumption and the number of Kubernetes object per namespace.  
Without ResourceQuotas, a single namespace can consume all cluster resources and starve other workloads.  
It avoids chaos in the cluster and brings peace between teams.  

## Why you need ResourceQuotas

If you have multi tenancy clusters in production, this common issues will sound familiar:  

- **The noisy neighbor problem:** Team A deploys an “hungry” application that consumes most of the cluster capacity, then Team B can't deploy anymore.
- **The unlimited deployment:** A nasty typo and someone deployed 1000 replicas instead of 10, the scheduler might taking down other workloads to make room.
- **The forgotten test environment:** A stress test deployment with high resource requests is deployed and then everyone forget about it but it still consume resources

`ResourceQuotas` prevent all of these scenarios by enforcing hard limits at the namespace level.

## ResourceQuota manifest

Let's create a basic ResourceQuota for a namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dummy-quota
  namespace: dummy-ns
spec:
  hard:
    requests.cpu: "1"
    requests.memory: "2Gi"
    limits.cpu: "2"
    limits.memory: "4Gi"
    pods: "10"
```

This quota will enforce in the namespace `dummy-ns` the following:

- Total CPU requests **cannot exceed 1 core**
- Total memory requests **cannot exceed 2Gi**
- Total CPU limits **cannot exceed 2 cores**
- Total memory limits **cannot exceed 4Gi**
- A **maximum of 10 pods** can be created in the namespace

You can then apply your manifest with `kubectl apply -f quota.yaml -n dummy-ns`.  
Now, if you try to create a deployment with memory request of 5Gi, it will be rejected by the API Server.  

The full list of what can be limited by a `ResourceQuota` is available at the [official documentation](https://kubernetes.io/docs/concepts/policy/resource-quotas/#types-of-resource-quota)

## LimitRange

Ok we created a ResourceQuota but a concern still remains: what if the manifest do not set resource requests/limits?  
If pods omit requests/limits, Kubernetes assumes zero, so `ResourceQuotas` don’t apply correctly.  
Our cluster peace is not yet achieved.  

The solution is to combine `ResourceQuotas` with `LimitRanges`.  
`LimitRange` is another Kubernetes object that focus on providing default values for pods that don't specify anything.  

Let’s take the following example:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dummy-ns
spec:
  limits:
  - default:
      cpu: 200m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
```

If we try to deploy a pod without resource requests/limits, Kubernetes will automatically apply these defaults:  

- pod memory will have a request of 128Mi and a limit of 256Mi
- Pod cpu will have a request of 100m and a limit of 200m
And so the ResourceQuota is enforced!  

There is still one point that we would like to prevent: a single pod taking all the quota.  
For that we will use another aspect of `LimitRange`, the `max` property:  

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dummy-ns
spec:
  limits:
  - max:
      cpu: 1
      memory: 1Gi
    type: Container
```

Now, a single pod cannot request more than 1Gi memory and 1 CPU core

**Note that the limitRange manifest has been split into 2 for clarity, but in your production it’s a unique resource**

## ResourceQuota usage

To see current usage, you can describe the quota (remember that quota is namespace scoped so -n argument is needed)

```bash
kubectl describe resourcequota dummy-quota -n dummy-ns
```

Output:

```bash
Name:            dummy-quota
Namespace:       dummy-ns
Resource         Used   Hard
--------         ----   ----
requests.cpu     0.5    1
requests.memory  1.5Gi  2Gi
limits.cpu       1      2
limits.memory    3Gi    4Gi
```

## Common pitfalls

- **Setting quotas too tight:** Teams constantly hit the limits, productivity suffers.

- **Forgetting LimitRanges:** Quotas are not enforced, always create LimitRange alongside ResourceQuota.

- **Quotas usage are not monitored:** Quotas are not set once and then forgotten, they have to be adjusted over usage and time

- **Limiting the wrong things:** Don't limit everything just because you can, focus on resources that matters for your production

## Conclusion

`ResourceQuotas` are essential for any multi-tenant Kubernetes cluster.  
They prevent resource hogging, ensure fair sharing, make capacity planning predictable.  
Combined with `LimitRanges` they bring peace to your clusters (and users)
