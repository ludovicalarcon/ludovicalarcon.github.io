---
layout: post
title: "Troubleshooting Kubernetes: ImagePullBackOff"
img: k8s-troubleshooting.png
tags: [kubernetes, throubleshooting, "101"]
---

## __Introduction__

In my day-to-day work, I help and support teams to shift on containers and cloud technologies, and I could observe that Kubernetes can be overwhelming for newcomers.  
One of the reasons of Kubernetes complexity is because troubleshooting what went wrong can by difficult if you don't know where to look and you need to often look in more than one place.  
I hope this series of blog posts about troubleshooting Kubernetes will help people in their journey.

- __Troubleshooting Kubernetes: ImagePullBackOff__
- [Troubleshooting Kubernetes: Pods in Pending State]({% post_url 2022-05-22-Troubleshooting-Pending-Pods %})
- [Troubleshooting Kubernetes: CrashLoopBackOff]({% post_url 2022-05-26-Troubleshooting-CrashLoopBackOff %})

## __Troubleshooting ImagePullBackOff Error__

You deployed your deployment (statefulset or other) and the underlying pod(s) is in `ImagePullBackOff` state.  
There can be 3 reasons for that:
- The `tag` does not exist
- The `image` does not exist
- You're using a `private registry` without credentials or wrong credentials

First, let's check the error message you have, for that you will use the `kubectl describe` command.  
This will provide you additional information on the pod, the output can be long. Let's jump to the `Events` section.

```sh
> kubectl get pod
nginx          0/1     ImagePullBackOff   0          17s
> kubectl describe pod nginx
```

## __Invalid Image Tag__

```sh
> kubectl describe pod nginx
Name:         nginx
...
...
Containers:
  nginx:
    Container ID:
    Image:          nginx:foo
    State:          Waiting
      Reason:       ImagePullBackOff
...
...
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  19s               default-scheduler  Successfully assigned dev/nginx to worker-node01
  Normal   BackOff    16s               kubelet            Back-off pulling image "nginx:foo"
  Warning  Failed     16s               kubelet            Error: ImagePullBackOff
  Normal   Pulling    6s (x2 over 18s)  kubelet            Pulling image "nginx:foo"
  Warning  Failed     5s (x2 over 17s)  kubelet            Failed to pull image "nginx:foo": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:foo": failed to resolve reference "docker.io/library/nginx:foo": docker.io/library/nginx:foo: not found
  Warning  Failed     5s (x2 over 17s)  kubelet            Error: ErrImagePull
```

Events with the `Failed Reason` are the ones you want to look at.
In this example, we got an interesting error message
```sh
Failed to pull image "nginx:foo": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:foo": failed to resolve reference "docker.io/library/nginx:foo": docker.io/library/nginx:foo: not found
```
This part is particularly interesting `nginx:foo: not found`. That indicates that the tag `foo` doesn't exist.  
You can easily check that on the docker hub or by pulling the image locally.

## __Invalid Image__

Another variant of the `ImagePullBackOff` error is the image doesn't exist.
```sh
> kubectl describe pod nginx
Name:         nginx
...
...
Containers:
  nginx:
    Container ID:
    Image:          nginxx
    State:          Waiting
      Reason:       ImagePullBackOff
...
...
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  42s                default-scheduler  Successfully assigned dev/nginx to worker-node01
  Normal   Pulling    25s (x2 over 41s)  kubelet            Pulling image "nginxx"
  Warning  Failed     24s (x2 over 40s)  kubelet            Failed to pull image "nginxx": rpc error: code = Unknown desc = failed to pull and unpack image "docker.io/library/nginxx:latest": failed to resolve reference "docker.io/library/nginxx:latest": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
  Warning  Failed     24s (x2 over 40s)  kubelet            Error: ErrImagePull
  Normal   BackOff    10s (x3 over 40s)  kubelet            Back-off pulling image "nginxx"
  Warning  Failed     10s (x3 over 40s)  kubelet            Error: ImagePullBackOff
```
We have a very similar output, but there is a slight difference in the error message.
```sh
Failed to pull image "nginxx": rpc error: code = Unknown desc = failed to pull and unpack image "docker.io/library/nginxx:latest": failed to resolve reference "docker.io/library/nginxx:latest": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
```
From that message you can identify two possible errors, either the `repository does not exist` or `you do not have access to it`.  
You need to glean another information now by answering this question `Am I using a private registry?`

If the answer is __no__, you have to double check the image name.  
But if you use a `private registry` we have to provide the credentials.

## __Private Registry__

In the case you are using a private registry, you have to provide the credentials to be able to pull the image.  
To do so you have to add the credentials as a `Kubernetes secret` and add the `imagePullSecrets` reference to it in your deployment (or other kubernetes objects that will create an underlying pod).

#### __Add the credential as a Kubernetes Secret__
```sh
kubectl create secret docker-registry my-registry-secret \
--docker-server=YOUR_REGISTRY_URL \
--docker-username=YOUR_USERNAME \
--docker-password=YOUR_PASSWORD
```
In the example the secret name is `my-registry-secret`.

### __Add the secret reference__

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  containers:
    - name: demo
      image: privaate/demo:v1.0
  # Reference to the secret
  imagePullSecrets:
    - name: my_registry-secret
```

For clarity purposes I used a pod in the example, but the same applies to deployment, statefulset and other.  
More information on [official documentation](https://kubernetes.io/docs/concepts/containers/images/#using-a-private-registry)

I hope this was helpful!