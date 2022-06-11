---
layout: post
title: "Troubleshooting Kubernetes: CrashLoopBackOff"
img: k8s-troubleshooting.png
tags: [kubernetes, throubleshooting, "101"]
---

## __Introduction__

In my day-to-day work, I help and support teams to shift on containers and cloud technologies, and I could observe that Kubernetes can be overwhelming for newcomers.  
One of the reasons of Kubernetes complexity is because troubleshooting what went wrong can by difficult if you don't know where to look and you need to often look in more than one place.  
I hope this series of blog posts about troubleshooting Kubernetes will help people in their journey.

- [Troubleshooting Kubernetes: ImagePullBackOff]({% post_url 2022-05-21-Troubleshooting-ImagePullBackOff %})
- [Troubleshooting Kubernetes: Pods in Pending State]({% post_url 2022-05-22-Troubleshooting-Pending-Pods %})
- __Troubleshooting Kubernetes: CrashLoopBackOff__

## __Troubleshooting Pods in CrashLoopBackOff State__

You deployed your deployment (statefulset or other) and the underlying pod(s) is in `CrashLoopBackOff` state.  
But what does this mean?
This means your pod is in a `starting, crashing` loop.

First, let's use the `kubectl describe` command.  
This will provide additional information to you on the pod. The output can be long, so let's jump to the `Events` section.

```sh
> kubectl get pod
NAME           READY   STATUS             RESTARTS      AGE
crashing-pod   0/1     CrashLoopBackOff   2 (17s ago)   48s
> kubectl describe pod crashing-pod
```

## __Application Misbehaving__

```sh
> kubectl describe pod crashing-pod
...
...
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  23s                default-scheduler  Successfully assigned default/crashing-pod to worker-node01
  Normal   Pulled     23s                kubelet            Successfully pulled image "alpine" in 875.741934ms
  Normal   Pulling    18s (x2 over 24s)  kubelet            Pulling image "alpine"
  Normal   Created    17s (x2 over 23s)  kubelet            Created container crashing-pod
  Normal   Started    17s (x2 over 23s)  kubelet            Started container crashing-pod
  Normal   Pulled     17s                kubelet            Successfully pulled image "alpine" in 1.300456734s
  Warning  BackOff    12s                kubelet            Back-off restarting failed container
```
From the events section, we can see that the pod was created, started and then it falls into this `BackOff` state.
```sh
Warning  BackOff    12s                kubelet            Back-off restarting failed container
```
This tells us that the container fails and Kubernetes is restarting it.  
This most likely means your container exited. As you may know, the container should keep `pid 1 running` or it will exit.  
Kuberntes will keep trying to restart it on an `Exponential backoff` model. You can see the `Restart` counter value increase as Kubernetes restarts the container.
```sh
> kubectl get pods
NAME           READY   STATUS             RESTARTS        AGE
crashing-pod   0/1     CrashLoopBackOff   6 (3m47s ago)   10m
```

At this point, we know our pod is stuck in a starting, exiting loop, let's now get the logs of the pod.
```sh
> kubectl logs crashing-pod
Hello World
Something bad is happening...
```
To keep the example simple, we just ouptut some text and then exiting the container.  
However, on your real application you will hopefully have some useful logs to tell you why it is exiting or at least a clue.  
An application misbehaving is one of the most common answer for the `CrashLoopBackOff` state.

## __Liveness probe__

Another possibility is that the `Liveness` probe does not returning a successful status and make the pod crashing.  
You will have the same `CrashLoopBackOff` state but the `describe` command will give you a different output.

```sh
NAME                 READY   STATUS             RESTARTS      AGE
liveness-fails-pod   0/1     CrashLoopBackOff   4 (12s ago)   3m27s
```

Let's check the `Events` section:
```sh
...
...
Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  3m56s                  default-scheduler  Successfully assigned default/liveness-fails-pod to worker-node01
  Normal   Pulled     3m55s                  kubelet            Successfully pulled image "alpine" in 883.359685ms
  Normal   Pulled     3m17s                  kubelet            Successfully pulled image "alpine" in 857.272018ms
  Normal   Created    2m37s (x3 over 3m55s)  kubelet            Created container liveness-fails-pod
  Normal   Started    2m37s (x3 over 3m55s)  kubelet            Started container liveness-fails-pod
  Normal   Pulled     2m37s                  kubelet            Successfully pulled image "alpine" in 882.126701ms
  Warning  Unhealthy  2m29s (x9 over 3m53s)  kubelet            Liveness probe failed: Get "http://192.168.87.203:8080/healthz": dial tcp 192.168.87.203:8080: connect: connection refused
  Normal   Killing    2m29s (x3 over 3m47s)  kubelet            Container liveness-fails-pod failed liveness probe, will be restarted
  Normal   Pulling    119s (x4 over 3m56s)   kubelet            Pulling image "alpine"
```
This two events are the one we are looking for.
```sh
Warning  Unhealthy  2m29s (x9 over 3m53s)  kubelet            Liveness probe failed: Get "http://192.168.87.203:8080/healthz": dial tcp 192.168.87.203:8080: connect: connection refused
Normal   Killing    2m29s (x3 over 3m47s)  kubelet            Container liveness-fails-pod failed liveness probe, will be restarted
```
The `liveness` probe keeps failing, so Kubernetes is killing and restarting the pod.
This can happen when we have either incorrectly configured the liveness probe or our application is not working.
We should start by checking one and then the other.
```yaml
...
...
livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
...
...
```

I hope this was helpful!