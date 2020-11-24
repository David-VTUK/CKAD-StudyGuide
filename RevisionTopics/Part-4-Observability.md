# Observability

# Understand LivenessProbes and ReadinessProbes

The kubelet uses liveness probes to know when to restart a container. For example, liveness probes could catch a deadlock, where an application is running, but unable to make progress. Restarting a container in such a state can help to make the application more available despite bugs.

The kubelet uses readiness probes to know when a container is ready to start accepting traffic. A Pod is considered ready when all of its containers are ready. One use of this signal is to control which Pods are used as backends for Services. When a Pod is not ready, it is removed from Service load balancers.

Take the following example:

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - sleep 30; touch /tmp/healthy; sleep 60 
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5

```

This pod will sleep for 30 seconds, write to `/tmp/healthy` and then wait 60 seconds. The `readinessProbe` will cat this file to determine if the container is ready to accept traffic

`initialDelaySeconds` influences the number of seconds after the container has started before liveness probes are initiated.

`periodSeconds` Influences how often (in seconds) to perform the probe.

Applying this manifest will yield:

```
NAME            READY   STATUS              RESTARTS   AGE
liveness-exec   0/1     ContainerCreating   0          2s
liveness-exec   0/1     Running             0          3s
liveness-exec   1/1     Running             0          35s
```

Note the ~30 second gap between when the Pod is in a `running` state, and when the underlying container is `ready`. Only when the underlying containers are ready is when traffic will be sent to the Pod (ie when fronted by a `service` object).

The above example leveraged `exec` as the probe type. There are three types that can be used:

* exec
* httpGet - Perform a `http get` request. An example would be:
```
readinessProbe:
  httpGet:
    path: /ready
    port: 80
```

* tcpSocket - Establish a connection to the specified port

```
readinessProbe:
  tcpSocket:
    port: 80
```

Both `readiness` and `liveness` are of the same type - ie kind `probe` https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#probe-v1-core

To implement a `livenessProbe`, simply rename accordingly:

```
livenessProbe:
  tcpSocket:
    port: 80
```

And apply similar logic, using either `exec`, `httpGet`, or `tcpSocket` probe types to match your application logic.

# Understand Container Logging

Containers that log to stdout and stderr can have their logs extracted via kubectl:

```
kubectl logs nginx
```

Where “nginx” is the name of a Pod. However, we can increase the scope by supplying labels:

```
kubectl logs -l app=nginx --all-containers=true
```

This will return all logs from containers in Pods with a defined label “app=nginx”.

Leverage `kubectl describe pod` to get information pertaining to a specific pod.

```
kubectl describe pod nginx-65899c769f-2pgzk
```

This can be used for other K8s types - ie `deployment`, `service`, `node`, etc.

Note at the end there are a list of events:


```
Events:
 Type    Reason                 Age   From                    Message
 ----    ------                 ----  ----                    -------
 Normal  Scheduled              18m   default-scheduler       Successfully assigned nginx-65899c769f-2pgzk to k8s-worker-01
 Normal  SuccessfulMountVolume  18m   kubelet, k8s-worker-01  MountVolume.SetUp succeeded for volume "default-token-rbt5s"
 Normal  Pulling                18m   kubelet, k8s-worker-01  pulling image "nginx"
 Normal  Pulled                 18m   kubelet, k8s-worker-01  Successfully pulled image "nginx"
 Normal  Created                18m   kubelet, k8s-worker-01  Created container
 Normal  Started                18m   kubelet, k8s-worker-01  Started container
```
Specifying `--previous` will output the logs for the previous instance of the container in a pod if it exists

```
kubectl logs --previous nginx
```
