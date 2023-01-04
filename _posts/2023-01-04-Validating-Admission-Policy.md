---
layout: post
title: "Validating Admission Policies in Kubernetes 1.26"
img: k8s-logo.png
tags: [kubernetes]
---

Kubernetes 1.26 introduces the `Validating Admission Policies`.  

Validating Admission Policies, an in-process alternative to Validating Admission Webhooks using the [Common Expression Language](https://github.com/google/cel-spec) (`CEL`) to describe the policies.

### Why not just stick with Admission Webhooks ?

Because Admission Webhooks need to be developed, maintain and operate over time.  
You need to:

- Implement and maintain the binary to handle the admission request
- Deploy it with monitoring and alerting
- Handle ssl certificates life-cycle
- Plan production operation to upgrade/rollback it
- Etc...
    
It's a lot just for one so image you have several to handle...

That's why `Admission Policies` come to the rescue!  
It removes all this complexity by allowing you to define CEL expressions into the Kubernetes resources `ValidatingAdmissionPolicy`.  
Then, you just need to bind the policy to the appropriate resources through `ValidatingAdmissionPolicyBinding`.  

As of today, there is no `Mutating Admission Policy`.

## Experiment with Validating Admission Policy

### Prerequisites

- Have a kubernetes 1.26 cluster.
    As we are using an Alpha feature, I highly recommend to use a cluster for development purpose. I will use `kubeadm` for this article, but you can use what suits you.
- Enable the `ValidatingAdmissionPolicy` feature gate
    For kubeadm, it requires some works.

### Kubeadm enable feature gate

Ssh to the node(s) where core componants are running and edit the `/etc/kubernetes/manifests/kube-apiserver.yaml` manifest file.

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    ...
    ...
    - --feature-gates=ValidatingAdmissionPolicy=true
    - --runtime-config=admissionregistration.k8s.io/v1alpha1=true
```

This enables the feature and the API on the `api-server` component.  
Now we need to edit the `kubelet service`, add the following to the `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` service file.

```
--feature-gates=ValidatingAdmissionPolicy=true
```

So, it looks like

```sh
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --feature-gates=ValidatingAdmissionPolicy=true"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
...
...
```

Finally, we need to reload the config and restart the kubelet service.

```sh
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```


### Policies creation

To create a policy, we previously saw that we need two resources:
- A `ValidatingAdmissionPolicy` that defines the logic of the policy using `CEL`
- A `ValidatingAdmissionPolicyBinding` that connects the policy to a scope

Let's start with a simple example where we want to create a policy that limits the number of replicas at 2 maximum in dev namespace.
```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicy
metadata:
  name: "max-2-replicas-in-dev"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["deployments"]
  validations:
    - expression: "object.spec.replicas <= 2"
      message: "In dev deployment should use 2 replicas maximum"
```
The manifest above is articulate as follow:
- `failurePolicy`: Can be either `Ignore` or `Fail`, default value is Fail. 
- `resourceRules`: Specify on which resources and operations the policy should be applied
- `validations`: Define the CEL expressions that the resources must satisfy and the message that will be displayed in case of denied. Message field is optional.

According to the [official documentation](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/#validation-expression), CEL expressions have access to the contents of the Admission request/response through some variables
- `object` - The object from the incoming request. The value is null for DELETE requests.
- `oldObject` - The existing object. The value is null for CREATE requests.
- `request` - Attributes of the admission request.
- `params` - Parameter resource referred to by the policy binding being evaluated. The value is null if ParamKind is unset.

Now we have our policy, we need to create the `ValidatingAdmissionPolicyBinding`
```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "max-2-replicas-in-dev-binding"
spec:
  policyName: "max-2-replicas-in-dev"
  matchResources:
    namespaceSelector:
      matchLabels:
        env: dev
```
- `policyName` represents the policy we want to bind
- `matchResources` defines on which scope the policy is applied.
in this example, it will be applied on every namespace with the label `environment: dev`

So now, let's apply our policy
```yaml
# max-2-replicas-in-dev.yaml

apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicy
metadata:
  name: "max-2-replicas-in-dev"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["deployments"]
  validations:
    - expression: "object.spec.replicas <= 2"
      message: "In dev deployment should use 2 replicas maximum"
---
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "max-2-replicas-in-dev-binding"
spec:
  policyName: "max-2-replicas-in-dev"
  matchResources:
    namespaceSelector:
      matchLabels:
        env: dev
```
```sh
> kubectl apply -f max-2-replicas-in-dev.yaml
```

We can test and validate our policy
```yaml
# deploy.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: sandbox
  labels:
    env: dev
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: sandbox
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
```
```sh
> kubectl apply -f deploy.yaml
The deployments "nginx" is invalid: : ValidatingAdmissionPolicy 'max-2-replicas-in-dev' with binding 'max-2-replicas-in-dev-binding' denied request: In dev deployment should use 2 replicas maximum
```
We can see the policy denied the deployment because we try to set 3 replicas. The message used is also the one we defined in the policy.  

If we don't set a message, we get the following output
```sh
The deployments "nginx" is invalid: : ValidatingAdmissionPolicy 'max-2-replicas-in-dev' with binding 'max-2-replicas-in-dev-binding' denied request: failed expression: object.spec.replicas <= 2
```

### Another one

Now let's say we want a policy to deny a deployment if it does not use an image with a tag.  
Our policy will look like that:
```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicy
metadata:
  name: "force-tagged-image"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["deployments"]
  validations:
    - expression: "object.spec.template.spec.containers.all(e, e.image.contains(':'))"
      message: "Image should use a tag"
---
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "force-tagged-image-binding"
spec:
  policyName: "force-tagged-image"
  matchResources:
```
You can see that we did not define a `matchResources` section like in our first example.  
This allows us to define a cluster-wide scope, it will apply be applied to every namespace.

Let's try our policy
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: sandbox
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.23.3
      - name: foo
        image: foo
```
```sh
> kubectl apply -f deploy.yaml
The deployments "nginx" is invalid: : ValidatingAdmissionPolicy 'force-tagged-image' with binding 'force-tagged-image-binding' denied request: Image should use a tag
```

It's possible to chain multiple expressions to create more complex validations.  
We might want to use exclusively a private registry and no latest tag for our prod.
```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicy
metadata:
  name: "force-tagged-image"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["deployments"]
  validations:
    - expression: "object.spec.template.spec.containers.all(e, e.image.contains(':'))"
      message: "Image should use a tag"
	  - expression: "object.spec.template.spec.containers.all(e, e.image.startsWith('myregistry.com'))"
      message: "Image should come from myregistry.com"
---
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "force-tagged-image-binding"
spec:
  policyName: "force-tagged-image"
  matchResources:
    namespaceSelector:
      matchLabels:
        env: prod
```

## To infinity and beyond …

You can find some examples of [Validation expression](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/#validation-expression-examples)  
And of course go through the [official documentation](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy)

With the new `ValidatingAdmissionPolicy` feature in k8s 1.26, we can now create custom powerful and/or complex policies to enforce security and compliance without the burdensome of managing `AdmissionWebhooks`.  
Of course, there is powerfull solutions like Kyverno or OPA with a lot of features, but ValidatingAdmissionPolicy is more lightweight and easier.  

Depending of your requirement you have now the choice between two approaches.

I hope you found that article helpful and enjoy while creating your own policies!