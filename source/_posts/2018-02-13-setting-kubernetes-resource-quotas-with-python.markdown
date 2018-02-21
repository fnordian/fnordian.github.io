---
layout: post
title: "Setting kubernetes resource quotas with python"
date: 2018-02-13 10:27:53 +0700
comments: true
categories: 
---
In an RBAC based kubernetes system, users' access to the cluster can be limitted using namespaces, roles and rules. 
These limits consists of resource-types and methods/verbs a user can apply on those. 
E.g. a user may create, list and delete pods.

While these limits already enable a quite effective isolation in a way, 
that one user may not modify the resources of another user (or the system), it is often necessary to constrain usage even more. 

So here is another tool to further control cluster usage: *resource quotas*.
Resource quotas let you limit the following resources on a per user-basis. 

* pods
* services
* replicationcontrollers
* resourcequotas
* secrets
* configmaps
* persistentvolumeclaims
* services.nodeports
* services.loadbalancers

You can find more documentation [here](https://kubernetes.io/docs/tasks/administer-cluster/quota-api-object/)

Let's see how to create and apply them with python: 

```python
import kubernetes
v1 = kubernetes.client.CoreV1Api()

resource_quota = kubernetes.client.V1ResourceQuota(
        spec=kubernetes.client.V1ResourceQuotaSpec(
            hard={"requests.cpu": "1", "requests.memory": "512M", "limits.cpu": "2", "limits.memory": "512M", 
                "requests.storage": "1Gi", "services.nodeports": "0"}))
resource_quota.metadata = kubernetes.client.V1ObjectMeta(namespace="user-namespace",
        name="user-quota")
v1.create_namespaced_resource_quota("user-namespace", resource_quota)
    
```

As with roles, resource quotas are applied to namespaces. So to set limits for a user, 
the quotas have to be configured with the user's namespace.

