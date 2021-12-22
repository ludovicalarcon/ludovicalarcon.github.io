---
layout: post
title: Kubernetes RBAC
img: k8s-logo.png
tags: [kubernetes]
---
# __What's RBAC in Kubernetes ?__

RBAC stands for `Role-Based Access Control` and to quote Kubernetes documentation:
> Role-based access control (RBAC) is a method of regulating access to computer or network resources based on the roles of individual users within your organization

So RBAC in Kubernetes is the way that you allow/restrict access to resources within the cluster.

# __How that works ?__

There is 3 elements involved in RBAC mechanism:
- __Subjects__: A user, group of users or service account
- __Resources__: Kubernetes object within the cluster (Pods, Deployments, Services, etc..)
- __Verbs__: Operations that can be executed to the resources (get, create, delete, etc..). In the end all of them are `CRUD` operations (Create, Read, Update, Delete)

Access to the `API Server` is controlled through `authentication`, `authorization` and `admission control`.

#### ![](/assets/images/2021-12-22-api-server-flow.png)

During the `authorization` phase, the api server will check if the `user` currently authenticated can perform the `operation` over the `resource` asked.

RBAC is only about adding permissions as they are `denied by default`.

Basically RBAC allows to specify which `operation` can be executed over a set of `resources` for a given `user`.

# __Why authorization matter ?__

Well, obviously this is about security but also keep in mind to follow theses two principle:
- `Least privilege principle`: __The users and programs should only have the necessary privileges to complete their tasks__
- `Separation of duty principle`: __Restricts users from getting more privileges than needed, with the aim of preventing fraud and errors__

If you have secured your accesses, you have gone a good way in the security journey.

# __RBAC objects__

RBAC objects are available since Kubernetes 1.8 and use the `rbac.authorization.k8s.io` API group.
The RBAC API declares four kinds of Kubernetes objects: `Role`, `ClusterRole`, `RoleBinding` and `ClusterRoleBinding`.

## __Role and ClusterRole__

`Role` and `ClusterRole` objects are used to define a set of permissions. Permissions are represented as a list of rules combining `apiGroups`, `resource types` and `verbs`.

A `Role` is scoped to a namespace, so the set of permissions is applied to the given namespace.

By contrast, a `ClusterRole` is a cluster-wide resource, so the set of permissions is applied across the entire cluster.

Role example from the [Kubernetes documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-example) that grant read access to pods in the _default_ namespace.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

For a `ClusterRole`, you just need to use the kind `ClusterRole` and omit the namespace as they are not namespaced.

## __RoleBinding and ClusterRoleBinding__

`RoleBind` and `ClusterRoleBindind` are the objects used to link the set of permissions defined in a `Role` or `ClusterRole` to `subjects` (user, group or service account).
Both holds a list of subjects and a reference to the role being granted.  
`RoleBinding` is used to grant permission within a specific namespace whereas `ClusterRoleBindings` grant that access to all the namespaces.  
A `RoleBinding` should refer to a `Role` in the same namespace but also refers to a `ClusterRole` and bind it to the namespace of the RoleBinding.

RoleBinding from the [Kubernetes documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#rolebinding-example) that grand the _pod-reader_ role defined previously to the user _Jane_ in the _default_ namespace.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# You can specify more than one "subject"
- kind: User
  name: jane # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

For a `ClusterRoleBinding`, you need to use the kind `ClusterRoleBinding`, omit the namespace and use `ClusterRole` as reference.

<br>

The main idea behind RBAC is to provide access to resources for subjects who require it.  
Always keep in mind the `least privilege principle`.