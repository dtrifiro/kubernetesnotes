# Kubernetes #

What is Kubernetes?
A way of managing sets of containers across a range of nodes.
## Concepts

### Cluster
A cluster in kubernetes is a set of hosts. One of these hosts is designed as a
'master' node which runs:

  - `kube-apiserver`
  - `kube-controller-manager`
  - `kube-scheduler`

which controls the other nodes and manages the state of the cluster. You control
the master using `kubectl`

All the other nodes run:

 - `kubelet`, which communicates with the master node
 - `kube-proxy`, a network proxy which reflects kubernetes networking services on each node

these nodes are responsible to run your application.

![kubecluster](https://cdn-images-1.medium.com/max/1600/1*KoMzLETQeN-c63x7xzSKPw.png)


#### References

[Cluster Administration Overview]: https://kubernetes.io/docs/concepts/cluster-administration/cluster-administration-overview/
1. [Kubernetes official docs on kubernetes cluster][Cluster Administration Overview]

### Node
A node is a worker machine in Kubernetes, previously known as a minion. A node may be a VM or physical machine, depending on the cluster. Each node contains the services necessary to run pods and is managed by the master components. The services on a node include the container runtime, `kubelet` and `kube-proxy`. See The Kubernetes Node section in the architecture design doc for more details.

#### References
[nodes overview]: https://kubernetes.io/docs/concepts/architecture/nodes/
1. [Kubernetes official docs on nodes][nodes overview]

### Pod
A [Pod][pod definition] is group of one or more containers with shared storage/network, and a specification for how to run the containers.

- Containers within a Pod share an IP address and port space, and can find each other via localhost: they use the same network namespace and/or use other methods of isolation (cgroups, etc).
- Applications within a Pod have access to shared *volumes* (directories containing data).
- Pods **aren’t intended to be treated as durable entities**. They won’t survive scheduling failures, node failures, or other evictions, such as due to lack of resources, or in the case of node maintenance. If the pod is deleted, the volume is also destroyed.

In terms of Docker, a pod is a group of containers with shared namespaces and shared filesystem voluems.

![Pod Diagram](https://d33wubrfki0l68.cloudfront.net/aecab1f649bc640ebef1f05581bfcc91a48038c4/728d6/images/docs/pod.svg)


#### References
[pod definition]: https://kubernetes.io/docs/concepts/workloads/pods/pod/
1. [Kubernetes official docs on Pods][pod definition]
2. [Distributed system toolkit patterns](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/)
3. [Why not multiple programs in a single docker container?] (https://kubernetes.io/docs/concepts/workloads/pods/pod/#alternatives-considered)


--------

### Volumes
#### In Docker
In Docker, a [volume][Docker Volumes] is simply a directory on disk or in another Container.

- Why? Volumes' size does not affect the size of the containers using it, and the volume’s contents exist outside the lifecycle of a given container.
- Docker automatically manages volumes (vs. bind mounts, in which the path(s) have to exist already).
- Can be shared across volumes.
- Can be stored on a remote host ([Volume Drivers](https://docs.docker.com/storage/#use-a-volume-driver)). For example, one can use a sshfs or NFS volume driver to store a volume on a remote host.

#### References
[Docker Volumes]: https://docs.docker.com/storage/volumes/
1. [Docker official docs on Volumes][Docker Volumes]


#### In Kubernetes
On-disk files inside a container are lost whenever the container is stopped. If the container crashes Kubernetes will restart it, but files will be lost. When running multiple containers inside a pod, we need a method to share files. A *volume* does exactly that.

The backend storage for volumes (called *Volume Plugins*) can use a wide range of technologies, such as:

- local storage
- glusterfs
- NFS
- iSCSI
- CSI (Container Storage Interface) defines a standard interface to expose arbitrary storage systems to their container workloads.
- Raw block volume (beta in 1.13)
- - ... ([lots more][list of volume backends])

These can be extended by other plugins.

---
##### Persistent Volumes
![cluster_pv_diagram](https://cdn-images-1.medium.com/max/1200/1*kF57zE9a5YCzhILHdmuRvQ.png)

- Persistent Volume (PV): a resource in the kubernetes cluster, just as a node of the cluster.
- Persistent Volume Claim (PVC): a request for storage by the user, similar to a POD, it consumes node resources (just like a Pod).

PVs can be provisioned statically or dynamically.

- Static: a fixed amount of PVs available on the cluster, availble for consumption.
- Dynamic: when a user requests a PVC (*PersistentVolumeClaim*), and there's no available PVs that match the request, the cluster tries to dynamically provision a volume for the PVC.

Important points:

- PVs and PVCs can have a *storage class* (see below)
- Reclaim policies can be defined for dynamically allocated PVs (this means PVCs?), such as retain, recycle (empty the volume and re-use it) and delete (delete the associated storage asset)
- Access Modes for each PV can be defined (ReadWriteOnce, ReadOnlyMany, ReadWriteMany), each PV plugin has different access mode capabilities.
- Mount options
- Node affinity: (?)

---

##### StorageClass
A StorageClass provides a way for administrators to describe the “classes” of storage they offer. Different classes might map to quality-of-service levels, or to backup policies, or to arbitrary policies determined by the cluster administrators

Usecase: NVMe storage vs SSD storage vs HDD storage, useful for providing different tiering levels?

StorageClasses can have different backends (provisioners) to provision PVs, such as:

- Local Storage
- Glusterfs
- NFS
- iSCSI
- external provisioners ([write your own provisioner][kubernetes-incubator/external-storage])
-  ... [lots more][list of storageclass provisioners]

 Reclaim policies can be set for dynamically created volumes.



#### References
[list of volume backends]: https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes
1. [Kubernetes official docs on Volumes][list of volume backends] "List of volumes backends"
[list of storageclass provisioners]: https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner
2. [Kubernetes official docs on Storage Classes][list of storageclass provisioners] "List of storageclass provisioners"
[kubernetes-incubator/external-storage]: https://github.com/kubernetes-incubator/external-storage
3. [kubernetes-incubator/external-storage] "library to write external provisioners"
[node affinity]: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#volumenodeaffinity-v1-core
4. [Kubernetes official docs on node affinity][node affinity]


### ReplicaSet
A ReplicaSet’s purpose is to maintain a stable set of replica Pods running at any given time. It is defines:

- Pods it can acquire
- Number of Pods it should be maintaining
- Pod template specifying data of new Pods it should create to match the specified number of replicas.

ReplicaSets should almost never used directly, use a Deployment to manage ReplicaSets.

### Deployment
A deployment manages ReplicaSets and provides declarative updates (?) to Pods

## Stateful Sets
Quoting from `https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/`

> Like a Deployment , a StatefulSet manages Pods that are based on an identical container spec. Unlike a Deployment, a StatefulSet maintains a sticky identity for each of their Pods.
> These pods are created from the same spec, but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling.

StatefulSets are valuable for applications that require one or more of the following.

- Stable, unique network identifiers.
- Stable, persistent storage.
- Ordered, graceful deployment and scaling.
- Ordered, automated rolling updates.


## Practical stuff
Define a deployment `yml` file. Here's an example for nginx:

    apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
     kind: Deployment
     metadata:
       name: nginx-deployment
     spec:
       selector:
         matchLabels:
           app: nginx
       replicas: 2 # tells deployment to  run 2 pods matching the template
       template:
         metadata:
           labels:
             app: nginx
         spec:
           containers:
           - name: nginx
             image: nginx:1.7.9
             ports:
             - containerPort: 80

Get the deployment status from the command line:

    kubectl get deployments


To access, it *probably* needs either an "ingress" (access) or a "LoadBalancer" service (how to define this?)

---

### Using volume(s):

1. `spec.volumes` defines the volume(s)
2. `.spec.containers.volumeMounts` defines where to mount the volume(s) in the Containers

Subpath: one volume per multiple uses in a single pod (https://kubernetes.io/docs/concepts/storage/volumes/#using-subpath)

- `volumeMounts.subPath` property

#### Persistent Volume Claims usage ####
Example of an application with a PVC

    apiVersion: v1
    kind: Pod
    metadata:
      name: mypod
    spec:
      containers:
        - name: myfrontend
          image: nginx
          volumeMounts:
          - mountPath: "/var/www/html"
            name: mypd
      volumes:
        - name: mypd
          persistentVolumeClaim:
            claimName: myclaim

### Example
Deploying wordpress and mysql using persistent volumes
https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/


### Stateful Sets
Quoting from `https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/`

> Like a Deployment , a StatefulSet manages Pods that are based on an identical container spec. Unlike a Deployment, a StatefulSet maintains a sticky identity for each of their Pods.
> These pods are created from the same spec, but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling.

StatefulSets are valuable for applications that require one or more of the following.

- Stable, unique network identifiers.
- Stable, persistent storage.
- Ordered, graceful deployment and scaling.
- Ordered, automated rolling updates.

---

## Hands-on

### Kubernetes dashboard

From the kubernetes master:

   $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml


Now run `kubectl proxy`. The kubernetes dashboard will be available at:

    http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

