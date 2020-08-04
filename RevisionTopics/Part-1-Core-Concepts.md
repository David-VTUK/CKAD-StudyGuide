# Understand Kubernetes API primitives

 Quite a large subject to cover, but is well documented at [https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)

# Create and configure basic Pods

Pods are the smallest deployable units of computing that you can create and manage in Kubernetes.

A Pod (as in a pod of whales or pea pod) is a group of one or more containers , with shared storage/network resources, and a specification for how to run the containers. A Pod's contents are always co-located and co-scheduled, and run in a shared context. A Pod models an application-specific "logical host": it contains one or more application containers which are relatively tightly coupled. In non-cloud contexts, applications executed on the same physical or virtual machine are analogous to cloud applications executed on the same logical host.

![K8s Pod Anatomy](./Images/pods.svg)
Source https://kubernetes.io/docs/tutorials/kubernetes-basics/explore/explore-intro/


There are a number of approaches to create a pod spec within K8s.

## Pod Anatomy

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

## Creating manifests from Scratch

This requires an intimiate knowledge of the Kubernets API spec for pods - https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#pod-v1-core

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

## Create manifests from a template

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

## Pod topology - Single container

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
## Pod topology - Multiple containers

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
  restartPolicy: Never
  containers:
  - name: nginx-container
    image: nginx
  - name: debian-container
    image: debian
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
```

This example also makes use of the `command` and `args` definition. In this example, it's to prevent the debian container from crashing, as all containers need to run a process.