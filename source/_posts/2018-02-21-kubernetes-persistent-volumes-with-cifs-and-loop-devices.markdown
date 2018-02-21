---
layout: post
title: "Kubernetes Persistent Volumes with CIFS and loop devices"
date: 2018-02-21 11:14:16 +0700
comments: true
categories: 
---
Here's an idea for a poor man's NAS for Kubernetes PVs: All you need is a fileserver running samba and some bash scripting.

But let's begin at how Kubernetes handles storage. The key philosophy here is, that storage is something different than 
computation, so storage get's it's own abstraction (i.e. Kubernetes resource).
While computation is basically handled by pods, persistent storage is offered by Persistent Volumes (PV), 
which can be mounted inside a pod's container as a volume. In order to mount a PV in a pod, 
the pod has to reference a Persistent Volume Claim (PVC).

The Persistent Volume Claim defines the requirements for the storage, e.g. capacity, and acts like a bridge between pod and
PV, bringing them together. After a PVC is deployed it is *pending* and the controller-manager tries to match it to a PV.
When that succeeds, the PVC is *bound* to the PV and available to be mounted in a pod as a volume.

{% mermaid %}
graph LR;
    pvc[Persistent Volume Claim] -- controller manager binds --> pv[Persistent Volume]
    pod[Pod] -- mounts --> pvc  
{% endmermaid %}

PVCs have a lifecycle separate from pods, so a pod can be created, deleted and recreated, while the PVC and with it the
persistent storage (the stored data) stays alive and *bound*. When the PVC gets deleted, the stored data becomes 
unavailble for good and the PV can be [reclaimed](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#reclaiming).

pod.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
    volumeMounts:
            - name: my-pv
              mountPath: /volume
  restartPolicy: Always

  volumes:
  - name: my-pv
    persistentVolumeClaim:
      claimName: my-pv-claim

```

pvc.yaml
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi 
```

Because PV is only an abstract concept of storage, there has to be something, that implements how files are persisted.
This is the storage *provisioner*. It's job is to mount some kind of storage on the so it's available for the pod. 
There are [several provisioners](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes) shipped within Kubernetes, most of them for proprietary cloud providers or datacenter-scale
storage systems. Luckily there is also **FlexVolume**, which allows you to implement your own provisioner easily.

pv.yaml
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  flexVolume:
    driver: "fnordian/cv"
    readOnly: false
    options:
      source: "//192.168.121.225/kubvolumes"
      mountOptions: "dir_mode=0700,file_mode=0600"
      cifsuser: "nobody"
      cifspass: "nobody"

```

Using FlexVolume
----------------

So FlexVolume does not implement any storage mounting itself, but it gives you the opportunity to do it yourself
in form of a *driver*.
The *driver*'s API is not too complicated and often implementing 3 actions (init, mount, unmount) is enough.

The details of a driver are explained [here](https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md).
The gist of creating a most basic driver is: It needs to have a name (vendor-drivername) and must reside as an executable
in a certain path on every node. It defaults to:

/usr/libexec/kubernetes/kubelet-plugins/volume/exec/*vendor*~*drivername*/*drivername*

The driver is then called with command line arguments, specifying the action and is expected to respond with json written to
stdout. Here is an example

```bash
/usr/libexec/kubernetes/kubelet-plugins/volume/exec/fnordian~cv/cv init

# response: {"status": "Success", "capabilities": {"attach": false}}

/usr/libexec/kubernetes/kubelet-plugins/volume/exec/fnordian~cv/cv mount \
    /var/lib/kubelet/pods/ce4b5af0-16de-11e8-bcfd-5254002cac04/volumes/fnordian~cv/pv0003 \
    {"cifspass":"nobody","cifsuser":"nobody","kubernetes.io/fsType":"","kubernetes.io/pod.name":"busybox",\
    "kubernetes.io/pod.namespace":"default","kubernetes.io/pod.uid":"ce4b5af0-16de-11e8-bcfd-5254002cac04",\
    "kubernetes.io/pvOrVolumeName":"pv0003","kubernetes.io/readwrite":"rw","kubernetes.io/serviceAccount.name":"default",\
    "mountOptions":"dir_mode=0700,file_mode=0600","source":"//192.168.121.159/kubvolumes"}

# response: {'status': 'Success'}

/usr/libexec/kubernetes/kubelet-plugins/volume/exec/fnordian~cv/cv unmount /var/lib/kubelet/pods/a80184b0-16df-11e8-bcfd-5254002cac04/volumes/fnordian~cv/pv0003

# response: {'status': 'Success'}
```

As you can see, the driver's mount action gets passed a target directory and parameters specifying the volume as JSON. 
After it returned with success, it is expected to have mounted a volume to the target directory. Unmounting follows the
same principle but only needs the target directory as the argument.

Mount storage on CIFS
---------------------

Now that we have the kubernetes specifics out of the way, all that's left is setting up a cifs-share and writing the driver. 
The idea for the driver is, that the actual volume is a filesystem inside a plain file, which is made available to the nodes via cifs and mounted for 
the pod using loop.

I agree, that it is a bit complex, but the advantage of a filesystem inside a loop-device over 
e.g. a cifs-share mounted directly to the pod is, that it can 
store files for multiple users (with different uids) and also selinux context labels.

You can find the running driver on [github](https://github.com/fnordian/fs-on-cifs-flexvolume). 
It expects name, username and password of a cifs-share as arguments, so be sure to have specified them in the PV.
Please note the lack of error- and corner-case-handling.

Setting up the cifs share with samba is also straight forward. I've included a sample smb.conf in the repository. 
Be sure to also set a password using smbpasswd.