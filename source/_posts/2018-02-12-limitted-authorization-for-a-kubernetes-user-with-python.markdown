---
layout: post
title: "Limitted authorization for a kubernetes user with python"
date: 2018-02-12 08:09:56 +0700
comments: true
categories: 
---
While last post's
was about [authenticating a user](/blog/2018/02/11/creating-client-certificates-for-kubernetes-apiserver-with-python/) 
to kubernetes, this one handles authorization.

Different levels of authorization in kubernetes can be achieved through *namespaces*, *roles* and *rolebindings*.
In order to limit a user to his/her own space in the cluster, we create a personal namespace and bind a role to it.
The role describes the types of access, the user will have.
This kind of authorization is called RBAC, [role-based access control](https://kubernetes.io/docs/admin/authorization/rbac/). 

A namespace in kubernetes is defined just by a name. So creating it is easy:
```python
from kubernetes import client
v1 = client.CoreV1Api()
v1.create_namespace("user-namespace")
```   

Roles are a bit more complex. They specify the access rights for the user. These rights are defined as a list of rules,
which are additive and declare the resources and the type of access the role will give. 

Besides roles, there are also cluster roles. The difference between those is, that a role is specific to a namespace,
while a cluster role is not. 

The example below gives a user enough rights to create, manage and delete deployments. Note that as a role is specific
to a namespace and that a user-namespace is specific to a user, there will be a role for every user.

```python
import kubernetes
rules = [
        kubernetes.client.V1PolicyRule([""], resources=["pods"], verbs=["get", "list", "create", "delete", "update"], ),
        kubernetes.client.V1PolicyRule(["extensions"], resources=["deployments", "replicasets"],
                                       verbs=["get", "list", "create", "delete", "update"], )
    ]
role = kubernetes.client.V1Role(rules=rules)
role.metadata = kubernetes.client.V1ObjectMeta(namespace="user-namespace",
                                               name="user-role")

rbac = kubernetes.client.RbacAuthorizationV1Api()
rbac.create_namespaced_role("user-namespace", role)
```

The created role must be bound to a user, otherwise it is not effective. Binding happens through role bindings.

```python
import kubernetes

role_binding = kubernetes.client.V1RoleBinding(
        metadata=kubernetes.client.V1ObjectMeta(namespace="user-namespace",
                                                name="user-role-binding"),
        subjects=[kubernetes.client.V1Subject(name="user", kind="User", api_group="rbac.authorization.k8s.io")],
        role_ref=kubernetes.client.V1RoleRef(kind="Role", api_group="rbac.authorization.k8s.io",
                                             name="user-role"))

rbac = kubernetes.client.RbacAuthorizationV1Api()                                             
rbac.create_namespaced_role_binding(namespace="user-namespace",
                                        body=role_binding)
```


And that's it. Now, the user 'user' has access to it's own namespace 'user-namespace' and may manage deployments.

There's one more thing. What if you want to make sure, a user does not deplete all the cluster's resources? 
With the current configuration, he or she is able to spawn new pods until all the nodes are full. 
It turns out, kubernetes has a way of handling this. It's called resource quota, and my next post will be about it.