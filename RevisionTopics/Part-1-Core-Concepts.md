# Understand Kubernetes API primitives

 Quite a large subject to cover, but is well documented at [https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)

# Create and configure basic Pods

Pods are the smallest deployable units of computing that you can create and manage in Kubernetes.

A Pod (as in a pod of whales or pea pod) is a group of one or more containers , with shared storage/network resources, and a specification for how to run the containers. A Pod's contents are always co-located and co-scheduled, and run in a shared context. A Pod models an application-specific "logical host": it contains one or more application containers which are relatively tightly coupled. In non-cloud contexts, applications executed on the same physical or virtual machine are analogous to cloud applications executed on the same logical host.

![K8s Pod Anatomy](./Images/pods.svg)
Source https://kubernetes.io/docs/tutorials/kubernetes-basics/explore/explore-intro/


There are a number of approaches to create a pod spec within K8s.

# Pod Anatomy

The specification for a pod can be quite simple, breaking down a sample Nginx pod spec reveals the following:

```
apiVersion: v1
kind: Pod
```

Like with any Kubernetes object, we have a `apiVersion` and `Kind`. `Pod` is its own kind

```
metadata:
  labels:
    run: nginx
  name: nginx
```

Next is metadata - in this example containing the name of the pod, and a label, which is formed of a key=value pair. 

```
spec:
  containers:
  - image: nginx
    name: nginx
```

Next is the meat of the pod spe. Which containers are in this pod, what name their given and what image they're based on. 

Putting it all together:

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
  restartPolicy: Always
```

# Creating manifests from Scratch

This requires an intimate knowledge of the Kubernetes API spec for pods - https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#pod-v1-core

To which a basic Pod spec can be created:

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
  restartPolicy: Always
```

# Create manifests from a template

This is a slightly more user friendly option and is likely helpful during the exam, leveraging `kubectl` to generate some YAML:

```
kubectl run nginx --image=nginx --dry-run -o yaml                                                          

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

# Pod topology - Single container

The below example highlights a single Pod with a single Container, but there are other Pod types.

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
  restartPolicy: Always
```
# Pod topology - Multiple containers

There are situations where having multiple containers in a pod is beneficial. Particularly in scenarios that require containers to be tightly coupled

```
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
  labels:
    run: nginxplusdebian
  name: nginxplusdebian
spec:
  containers:
  - name: nginx-container
    image: nginx
  - name: debian-container
    image: debian
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
```

This example also makes use of the `command` and `args` definition to prevent the debian container from crashing, as all containers need to run a process.


# Pod topology - Multiple containers with shared storage

Adding a `volume` stanza within the pod specification creates a volume of a certain type. As Pods share resources that are allocated to it, the volume can be mounted by both containers

```
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
  ```

  An `emptyDir` volume is first created when a Pod is assigned to a Node, and exists as long as that Pod is running on that node. As the name says, it is initially empty. Containers in the Pod can all read and write the same files in the emptyDir volume, though that volume can be mounted at the same or different paths in each Container. When a Pod is removed from a node for any reason, the data in the emptyDir is deleted forever.

  Other volume types include (but not limited to) AzureDisk, iSCSI, NFS, Local, Fibre Channel and cinder.

  Both containers mount the volume, `debian-container` writes into this directory, and `nginx` reads from it.

  # Init containers

  The previous examples leverage the `container` stanza in the pod specification. These are typically used to run an application. There are, however, other container types, `init` being one of them.

  `init`, or initalization containers are always executed before those declared with the `container` stanza. If an `init` container fails, the Pod is declared as failed.

  Reasons for using `init` containers normally include:

  * Preparing a volume by prepopulating (`git clone`)
  * Waiting for a service or external resource to be ready

  ```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-container
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  ```

  In this example, `init-container` will run until completion prior to `myapp-container` runs. The former will perform a `nslookup` every 2 seconds until it can resolve `myservice`. Once it can, the `init-container` will finish and `myapp-container` will run.


  # Ephemeral containers

Ephemeral containers may be run in an existing pod to perform user-initiated actions such as debugging. In order to add an ephemeral container to an existing pod, use the pod's ephemeralcontainers subresource. This field is alpha-level and is only honored by servers that enable the EphemeralContainers feature.

Whereas `initContainers` are used proactively, `ephemeralContainers` are more geared towards reactive, interactive troubleshooting. For example, a Pod can be spun up with a single container:

`kubectl run nginxtest --image=nginx`

"Inject" a debug container based on the busybox image into this Pod.

`kubectl alpha debug nginxtest -i --image=busybox`

The image selected for debugging will have all the necessary tooling, binaries, utilities etc to debug the application running in the Pod.