# Understand Services

Pods are ephemeral. Therefore placing these behind a service which provides a stable, static entrypoint is a fundamental use of the kubernetes service object. To reiterate, services take the form of the following:

* ClusterIP - Internal only
* LoadBalancer - External, requires cloud provider to provide one
* NodePort - External, requires access the nodes directly
* Ingress resource - L7 An Ingress can be configured to give services externally-reachable URLs, load balance traffic, terminate SSL, and offer name based virtual hosting. An Ingress controller is responsible for fulfilling the Ingress, usually with a loadbalancer, though it may also configure your edge router or additional frontends to help handle the traffic.

An example of deploying a Load Balancer:
```
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
	run: my-nginx
spec:
  type: LoadBalancer
  ports:
  - port: 80
	protocol: TCP
	name: http
	targetPort: 80
  selector:
	run: my-nginx
```

* Port- The port the LB itself will listen on

* Target port - The port on the pods to forward connections to

* Selector - The pods that we want to load balance to. In the above example pods matching the `run: my-nginx` label.

# Demonstrate Basic Understanding of `NetworkPolicies`

These determine if pods communicate with each other. By default this is left completely open. Think of this as firewall rules. We can apply network policies by:

*   Pod selectors (labels)
*   Namespace selectors
*   CIDR blocks ranges

If we were to run a nginx container and curl to it from a pod, it works as expected.

```
/home # curl 10.10.57.2
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
   body {
```


The following example creates a `NetworkPolicy` object that denies all inbound traffic in the default namespace


```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```


After applying and redoing the curl test, it fails:


```
/home # curl 10.10.57.2
curl: (7) Failed connect to 10.10.57.2:80; Connection timed out
```


A fleshed out network policy looks like the following:


```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```