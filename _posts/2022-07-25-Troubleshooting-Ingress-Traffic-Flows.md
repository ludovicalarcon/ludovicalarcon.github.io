---
layout: post
title: "Troubleshooting Kubernetes: Ingress Traffic Flows"
img: k8s-troubleshooting.png
tags: [kubernetes, throubleshooting, "101"]
---

## __Introduction__

In my day-to-day work, I help and support teams to shift on containers and cloud technologies, and I could observe that Kubernetes can be overwhelming for newcomers.  
One of the reasons of Kubernetes complexity is because troubleshooting what went wrong can by difficult if you don't know where to look and you need to often look in more than one place.  
I hope this series of blog posts about troubleshooting Kubernetes will help people in their journey.

- [Troubleshooting Kubernetes: ImagePullBackOff]({% post_url 2022-05-21-Troubleshooting-ImagePullBackOff %})
- [Troubleshooting Kubernetes: Pods in Pending State]({% post_url 2022-05-22-Troubleshooting-Pending-Pods %})
- [Troubleshooting Kubernetes: CrashLoopBackOff]({% post_url 2022-05-26-Troubleshooting-CrashLoopBackOff %})
- __Troubleshooting Kubernetes: Ingress Traffic Flows__

## __Troubleshooting Ingress Traffic Flows__

You deployed an ingress controller and you just deployed your api/app, the associated service and ingress resources.  
Finally you try to curl it, but unfortunately you're not able to reach your API endpoint.  
Let's see how you can troubleshoot it with an `inside out approach`.  
I generally use `inside out approach` because the issue is most likely with the api/app or the configuration. Especially if you already have other ingress traffic working.  

## __Traffic Flows__

First things first, what's the flow when you curl your api/app?
The process is the following:
```
curl http://www.example.com/whoami
  --> DNS Resolution
    --> Ingress Controller: host: www.example.com | path: /whoami
      --> whoami-svc:appPort
        --> one of the pods associated with the service
```
- DNS resolution happens and retrieves the IP address associated with the domain name
- The request goes to that IP address
- The request hits the ingress controller (Nginx, Traefik, ...)
- The ingress controller looks at the path and retrieve the associated service  
  In our case, the path is `/api` and the service is `my-api-svc`
- The service sends the request to one of the pods that it handles
- Your api/app handles the request

## __Inside Out Approach__

Now that we exactly know the flow of our request, let's start the inside out troubleshooting.  

### __The Pods__

The first things to do is check if the pod is in a `running` state.
```sh
> kubectl get pod -owide
NAME                                 READY   STATUS    RESTARTS   AGE    IP               NODE            NOMINATED NODE   READINESS GATES
whoami-78bbd86bbf-t4f4s              1/1     Running   0          113s   192.168.87.201   worker-node01   <none>           <none>
```
Then look at the logs to ensure everything looks good.
```sh
> kubectl logs whoami-78bbd86bbf-t4f4s
info: Microsoft.Hosting.Lifetime[0]
      Now listening on: http://[::]:5000
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /app
```

Finally, we will try to curl the pod directly with the help of `kubectl port-forward`
```sh
> kubectl port-forward whoami-78bbd86bbf-t4f4s 5000:5000
Forwarding from 127.0.0.1:5000 -> 5000
Forwarding from [::1]:5000 -> 5000
```
In another shell, you can now try to curl the pod and ensure request goes through via `localhost`.
```sh
> curl localhost:5000
Hostname: whoami-deployment-78bbd86bbf-t4f4s on IP: ::ffff:127.0.0.1
```

If you can reach your pod without issue, this means the problem is higher up.

### __The Service__

At the pod level, everything looks in order, let's now check the service.  
First, we will ensure that the service is correctly setup.
```sh
> kubectl get svc
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
whoami-svc                           ClusterIP      10.102.225.26    <none>        80/TCP                       2m

> kubectl describe svc whoami-svc
Name:              whoami-svc
Namespace:         dev
Labels:            app=whoami
Annotations:       <none>
Selector:          app=whoami
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.102.225.26
IPs:               10.102.225.26
Port:              80/TCP
TargetPort:        5000/TCP
Endpoints:         192.168.87.201:5000
Session Affinity:  None
Events:            <none>
```
There are 3 things you need to check:
- The `Selector` must match pod label
- The `TargetPort` must match the port you are using in your pod
- The `Endpoints` section must contain the IP address of your pod. We previously got it in the output of the pod `192.168.87.201`

Everything looks good, let's now try to curl our service.
```sh
> kubectl port-forward svc/whoami-svc 8080:80
Forwarding from 127.0.0.1:8080 -> 5000
Forwarding from [::1]:8080 -> 5000
```
And in another shell
```sh
> curl localhost:8080
Hostname: whoami-deployment-78bbd86bbf-t4f4s on IP: ::ffff:127.0.0.1
```

If you can reach your service without issue, this means the problem is higher up.

### __The Ingress__

Now we will looks into the Kubernetes Ingress.
```sh
> kubectl get ingress
NAME         CLASS   HOSTS         ADDRESS      PORTS   AGE
whoami-ing   nginx   example.com   10.0.0.241   80      2m

> kubectl describe ingress whoami-ing
Name:             whoami-ing
Labels:           <none>
Namespace:        dev
Address:          10.0.0.241
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host         Path  Backends
  ----         ----  --------
  example.com
               /whoami   whoami-svc:80 (192.168.87.201:5000)
Events:
  Type    Reason  Age                  From                      Message
  ----    ------  ----                 ----                      -------
  Normal  Sync    2m                   nginx-ingress-controller  Scheduled for sync
```

You need to ensure that the routing is correct:
- The `Host` is the one you're using
- The `Service` is the one we checked previously
- The `Port` is the same used in the service declaration

Now let's check the `Ingress Controller`.

### __Ingress Controller__

Check the logs of the ingress controller pod(s), you could find errors with a useful message to fix.
```sh
> kubectl logs ingress-nginx-controller-6bf7bc7f94-zgms5
```
Then `exec` on the pod to curl locally through the ingress controller.
As the host and the path are used to route traffic, you will need to specify both.
`curl -H "Host: example.com" localhost/whoami"
```sh
> kubectl exec -it ingress-nginx-controller-6bf7bc7f94-zgms5 -- sh
sh> curl -H "Host: example.com" localhost/whoami
Hostname: whoami-deployment-78bbd86bbf-t4f4s on IP: ::ffff:192.168.87.201
```

If you can reach your service without issue, this means the problem is higher up.

### __DNS__

If everything you check previously worked fine, the last thing you need to check is `DNS`.  
You need to ensure that the `DNS Resolution` gives the `external ip` of your `LoadBalancer` use by the ingress controller.  
In our case `10.0.0.241`.

You need to run `nslookup` on your local machine.
```sh
> nslookup example.com
Server:		127.0.0.53
Address:	      127.0.0.53#53

Name:	example.com
Address: 10.0.0.241
```

If it doesn't return the correct IP, then the DNS for “example.com” is set to an incorrect `CNAME` or the `CNAME` entry is not set.

By adding the IP address and the Host in your `/etc/hosts` file, you can make it works only for your local machine.  
That's how I'm able to get my IP with "example.com"

You now have all the steps to troubleshoot you `Ingress Traffic Flows`.

I hope this was helpful!