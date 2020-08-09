# Understand ConfigMaps

Configmaps are a way to decouple configuration from pod manifest file, this keeps our pod configuration more simple, and changes to configuration can be made outside of the Pod configuration.

**ConfigMaps should not be used to store confidential information.** Storing such information (such as API keys, passwords, etc) should be facilitated by `secrets` -which are covered later.

Configmaps take the format of key/value pairs and are usually leveraged for:

* Command line arguments to the entrypoint of a container
* Environment variables for a container
* Add a file in read-only volume, for the application to read
* Write code to run inside the Pod that uses the Kubernetes API to read a ConfigMap

The anatomy of a ConfigMap manifest is as so:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-demo
data:
  somekey: "somevalue"
```
But it can also be created with `kubectl`:

`kubectl create configmap vt-cm --from-literal=blog=virtualthoughts.co.uk`

And populated from a file:

`kubectl create configmap vt-cm --from-file=path/to/bar`


# Using a ConfigMap to populate an environment variable

By itself, a ConfigMap is somewhat meaningless. It needs to be mounted by a Pod. 

A simple example is to populate an environment variable using a value from the config map. The manifest below creates a configmap and the subsequent Pod uses it to populate the CONFIGMAP environment variable:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-demo
data:
  somekey: "somevalue"
---
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: cm-demo-pod
      image: nginx
      env:
        - name: CONFIGMAP 
          valueFrom:
            configMapKeyRef:
              name: cm-demo        
              key: somekey
```

An `exec` command can validate this has been set:

```
kubectl exec -it configmap-demo-pod -- env | grep CONFIG                         
CONFIGMAP=somevalue
```

# Mounting a configmap

ConfigMaps can be quite versatile. Instead of populating environment variables we can also mount them as files. 

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-demo
data:
  configfile.txt: |
    configSetting1=configValue1
    configSetting2=configValue2
    configSetting3=configValue3
    configSetting4=configValue4
---
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod-mount
spec:
  containers:
  - name: cm-demo-pod
    image: nginx
    volumeMounts:
    - name: configfile
      mountPath : /tmp
  volumes:
  - name: configfile
    configMap:
      name: cm-demo
```

The above will mount the configmap as a file in /tmp. A `exec` can verify this:

```
kubectl exec -it configmap-demo-pod-mount -- cat /tmp/configfile.txt          
configSetting1=configValue1
configSetting2=configValue2
configSetting3=configValue3
configSetting4=configValue4
```

# Understand SecurityContexts

A security context defines privilege and access control settings for a Pod **or** Container. `SecurityContexts` applied at the Pod level apply to all containers within that Pod. Alternatively, they can be set on specific containers.

Common use cases are to set items like:

* `runAsUser` and `runAsGroup` - The user and group ID's of the container
* `privileged` - Sets whether the Pod(s) - A "privileged" container is given access to all devices on the host. This allows the container nearly all the same access as processes running on the host. This is useful for containers that want to use linux capabilities like manipulating the network stack and accessing devices.

# Setting a SecurityContext at the Pod level

```
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1500
  containers:
  - name: sec-demo
    image: busybox
    command: [ "sh", "-c", "sleep 1h" ]
  - name: sec-demo2
    image: busybox
    command: [ "sh", "-c", "sleep 1h" ]
```

Shelling into either container in the Pod and performing a `ps` command reveals the userid of the process:

```
kubectl exec -it security-context-demo -c sec-demo -- ps 

PID   USER     TIME  COMMAND
    1 1500      0:00 sleep 1h
   11 1500      0:00 ps
```

```
kubectl exec -it security-context-demo2 -c sec-demo -- ps 
    
PID   USER     TIME  COMMAND
    1 1500      0:00 sleep 1h
   11 1500      0:00 ps
```

Performing the same but for a specific container:

```
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  containers:
  - name: sec-demo
    image: busybox
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
      runAsUser: 1500
  - name: sec-demo2
    image: busybox
    command: [ "sh", "-c", "sleep 1h" ]
```

Pod `sec-demo` runs as userID 1500

```
kubectl exec -it security-context-demo -c sec-demo -- ps  
PID   USER     TIME  COMMAND
    1 1500      0:00 sleep 1h
    6 1500      0:00 ps
```
Pod `sec-demo2` runs as user root

```
kubectl exec -it security-context-demo -c sec-demo2 -- ps 
PID   USER     TIME  COMMAND
    1 root      0:00 sleep 1h
    7 root      0:00 ps
```


# Define an application’s resource requirements

The Kubernetes Pod spec permits the configuration of memory and CPU requests and limits within the `resources` stanza.

A Container is guaranteed to have as much memory as it requests, but is not allowed to use more memory than its limit. 

A Container cannot use more CPU than the configured limit. Provided the system has CPU time free, a container is guaranteed to be allocated as much CPU as it requests.

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx-limits
spec:
  containers:
  - image: nginx
    name: nginx
    resources:
      limits:
        cpu: 1
        memory: "200Mi"
      requests:
        cpu: 0.5
        memory: "100Mi"
```

# Create and consume Secrets

To reiterate a previous point - **ConfigMaps should not be used to store confidential information.**


Secrets allow us to store and manage sensitive information pertaining to our applications, which can take the form of a variety of objects such as usernames, passwords, ssh keys and much more. 

Similarly to configmaps, secrets are designed to decouple this information directly from the pod declaration.

As part of a Kubernetes cluster stand up, secrets are already leveraged, and can be viewed them by executing the following:

`kubectl get secrets -A`

Secrets have three subtypes:

* Docker-Registry - for use with a docker registry
* Generic - created from a directory, file or literal
* TLS - for TLS certificates

Examples to create from `kubectl` :

`kubectl create secret tls tls-secret --cert=path/to/tls.cert --key=path/to/tls.key`

`kubectl create secret generic my-secret --from-literal=key1=supersecret`

The anatomy of a secret manifest is as so:

```
apiVersion: v1
data:
  key1: c3VwZXJzZWNyZXQ=
kind: Secret
metadata:
  name: my-secret
```

The contents of the secret key are **encoded** by default, **not encrypted**. To encrypt secrets at rest, a `EncryptionConfiguration` object needs to be present - more details of which can be found here: https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/

Consuming secrets is very similar to configmaps. (note as environment variables is one of a number of ways to leverage secrets)

To encode a string in Base64 in Linux, use the `Base64` command

```
echo "dbu" | base64

ZGJ1Cg==
```

```
apiVersion: v1
data:
  username: ZGJ1Cg==
  pass: ZGJwCg==
kind: Secret
metadata:
  name: my-secret
---
apiVersion: v1
kind: Pod
metadata:
 name: secret-test-pod
spec:
 containers:
 - name: secret-demo-pod
   image: nginx
   env:
     - name: DB_Username
       valueFrom:
         secretKeyRef:
           name: my-secret
           key: username
     - name: DB_Pass
       valueFrom:
         secretKeyRef:
           name: my-secret
           key: pass
```

```
kubectl exec -it secret-demo-pod -- env | grep DB_

DB_Username=dbu
DB_Pass=dbp
```

# Understand ServiceAccounts

A service account provides an identity for processes that run in a Pod, with this identity, Pods can interact with the Kubernetes API server.

Pods that are created without an explicit service account binding are automatically assigned the `default` service account in the namespace for which it resides in. For example:

```
kubectl get po nginx -o yaml | grep serviceAccount

serviceAccount: default
serviceAccountName: default
```
The default ServiceAccounts only has limited rights, consequently, it is generally best practice to create a ServiceAccount for each application, giving it minimal rights it requires.

The first step is to create a new Service Account:

```
apiVersion: v1
kind: ServiceAccount
metadata:
 name: serviceaccount-demo
```
Which, by itself, is not very helpful - it has no rights. For this, we need to create a `role` - which defines the permissions that objects bound to it will inherit:

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
 name: list-pods
 namespace: default
rules:
 — apiGroups:
   — ''
 resources:
   — pods
 verbs:
   — list
```

Next, we need to bind the `service account` to this `role` with a `role binding`

```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: default
subjects:
- kind: ServiceAccount
  name: serviceaccount-demo
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

```

And finally, configure a Pod to use this `Service Account` as defined i the `spec` stanza.

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-sa
spec:
  serviceAccountName: serviceaccount-demo
  containers:
  - image: nginx
    name: nginx
```