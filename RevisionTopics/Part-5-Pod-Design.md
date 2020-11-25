# Understand Deployments and how to perform rolling updates

Deployments are intended to replace Replication Controllers. They provide the same replication functions (through Replica Sets) and also the ability to rollout changes and roll them back if necessary. An example configuration is shown below:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

To update an existing deployment, we have two main options:

* Rolling Update
* Recreate

A rolling update, as the name implies, will swap out containers in a deployment with one created by a new image.

Use a rolling update when the application supports having a mix of different pods (aka application versions). This method will also involve no downtime of the service, but will take longer to bring up the deployment to the requested version

A recreate will delete all the existing pods and then spin up new ones. This method will involve downtime. Consider this a “bing bang” approach.

Examples listed in the Kubernetes documentation are largely imperative, but I prefer to be declarative. As an example, create a new yaml file and make the required changes, in this example, the version of the nginx container is incremented.


```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15
        ports:
        - containerPort: 80
```

We can then apply this file `kubectl apply -f updateddeployment.yaml --record=true`

```
kubectl rollout history deployment/nginx-deployment

deployment.extensions/nginx-deployment
REVISION  CHANGE-CAUSE
1     	<none>
2     	<none>
4     	<none>
5     	kubectl apply --filename=updateddeployment.yaml --record=true
```

Alternatively, we can also do this imperatively:

```
kubectl --record deployments/nginx-deployment set image deployments/nginx-deployment nginx=nginx:1.9.1

deployment.extensions/nginx-deployment image updated
deployment.extensions/nginx-deployment image updated
```

# Understand Deployments and how to perform rollbacks

To rollback to the previous version:

`kubectl rollout undo deployment/nginx-deployment`

To rollback to a specific version:

`kubectl rollout undo deployment/nginx-deployment --to-revision 5`

Where revision number comes from `kubectl rollout history deployment/nginx-deployment`

# Understand Jobs and CronJobs

A job in Kubernetes is a pod that executes a process that runs to completion. Examples include processing some data to produce an output, or performing a backup operation. `Jobs` are their own API type in Kubernetes, an example is listed below, which computes π to 2000 places and prints it out

```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```
To describe/extract logs from `Jobs`:

`kubectl describe jobs/pi`
`kubectl logs $PodName` 

A `CronJob` creates `Jobs` on a repeating schedule.

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: pi
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
            containers:
            - name: pi
              image: perl
              command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
            restartPolicy: Never
```