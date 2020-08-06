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

# Using a ConfigMap as a volume mount

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