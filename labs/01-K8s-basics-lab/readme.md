# Kubernetes Basics Lab

- [Kubernetes Basics Lab](#kubernetes-basics-lab)
  - [01. Run Application](#01-run-application)
    - [Objectives](#objectives)
    - [Creating and exploring an nginx deployment](#creating-and-exploring-an-nginx-deployment)
    - [Updating the deployment](#updating-the-deployment)
    - [Scaling the application by increasing the replica count](#scaling-the-application-by-increasing-the-replica-count)
    - [Deleting a deployment](#deleting-a-deployment)

## 01. Run Application

### Objectives

- Create an nginx deployment.
- Use kubectl to list information about the deployment.
- Update the deployment.

### Creating and exploring an nginx deployment 

You can run an application by creating a Kubernetes Deployment object, and you can describe a Deployment in a YAML file. For example, this YAML file describes a Deployment that runs the nginx:1.14.2 Docker image: [01-nginx-deployment.yml](01-nginx-deployment.yml).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
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

1. Create a Deployment based on the YAML file:

```sh
kubectl apply -f 01-nginx-deployment.yml
```

```sh
deployment.apps/nginx-deployment created
```

2. Display information about the Deployment:

```sh
kubectl describe deployment nginx-deployment
```

The output is similar to this:

```sh
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Thu, 27 Jan 2022 18:29:02 +0100
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.14.2
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-66b6c48dd5 (2/2 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  16m   deployment-controller  Scaled up replica set nginx-deployment-66b6c48dd5 to 2
```

3. List the Pods created by the deployment:

```kubectl
kubectl get pods -l app=nginx
```

The output is similar to this:

```sh
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-66b6c48dd5-cctwp   1/1     Running   0          19m
nginx-deployment-66b6c48dd5-hqvmj   1/1     Running   0          19m
```

4. Display information about a Pod:

Allocate a running nginx pod name to POD_NAME variable:

```sh
POD_NAME="$(kubectl get pods -l app=nginx | grep Running | head -1 | awk '{print $1}')"
```

Describe the Pod information:

```sh
kubectl describe pod $POD_NAME
```

The output is similar to this:

```sh
Name:         nginx-deployment-66b6c48dd5-cctwp
Namespace:    default
Priority:     0
Node:         ip-172-20-50-50.ec2.internal/172.20.50.50
Start Time:   Thu, 27 Jan 2022 18:29:02 +0100
Labels:       app=nginx
              pod-template-hash=66b6c48dd5
Annotations:  cni.projectcalico.org/containerID: 5e50be01e70e30f68b6ed1b2276e18230c6a8e020ac55f4b6170a87efdb569d7
              cni.projectcalico.org/podIP: 100.115.37.69/32
              cni.projectcalico.org/podIPs: 100.115.37.69/32
              kubernetes.io/limit-ranger: LimitRanger plugin set: cpu request for container nginx
Status:       Running
IP:           100.115.37.69
IPs:
  IP:           100.115.37.69
Controlled By:  ReplicaSet/nginx-deployment-66b6c48dd5
Containers:
  nginx:
    Container ID:   containerd://89a46e709d3a931ed6d7b67bea8d1b0add83834ba5a09e54e285dc07da758cdc
    Image:          nginx:1.14.2
    Image ID:       docker.io/library/nginx@sha256:f7988fb6c02e0ce69257d9bd9cf37ae20a60f1df7563c3a2a6abe24160306b8d
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thu, 27 Jan 2022 18:29:05 +0100
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:        100m
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-grwlg (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-grwlg:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>
```

### Updating the deployment 

You can update the deployment by applying a new YAML file. This YAML file specifies that the deployment should be updated to use nginx 1.16.1. [02-nginx-deployment-update](02-nginx-deployment-update.yml).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1 # Update the version of nginx from 1.14.2 to 1.16.1
        ports:
        - containerPort: 80
```

1. Apply the new YAML file:

```sh 
kubectl apply -f 02-nginx-deployment-update.yml
```

```sh
deployment.apps/nginx-deployment configured
```

2. Watch the deployment create pods with new names and delete the old pods:

```sh
kubectl get pods -l app=nginx
```

The output is similar to this:

```sh
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-559d658b74-4fpbt   1/1     Running   0          5m2s
nginx-deployment-559d658b74-8wzm8   1/1     Running   0          4m59s
```

### Scaling the application by increasing the replica count 

You can increase the number of Pods in your Deployment by applying a new YAML file. This YAML file sets replicas to 4, which specifies that the Deployment should have four Pods: [03-nginx-deployment-scale.yml](03-nginx-deployment-scale.yml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 4 # Update the replicas from 2 to 4
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1
        ports:
        - containerPort: 80
```

1. Apply the new YAML file:

```sh
kubectl apply -f 03-nginx-deployment-scale.yml
```

```sh
deployment.apps/nginx-deployment configured
```

2. Verify that the Deployment has four Pods:

```sh
kubectl get pods -l app=nginx
```

The output is similar to this:

```sh
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-559d658b74-h8mk5   1/1     Running   0          13s
nginx-deployment-559d658b74-ns28l   1/1     Running   0          13s
nginx-deployment-559d658b74-p9j85   1/1     Running   0          11s
nginx-deployment-559d658b74-pc5km   1/1     Running   0          11s
nginx-deployment-559d658b74-pkvqt   1/1     Running   0          13s
```

### Deleting a deployment 

1. Delete the deployment by name:

```sh
kubectl delete deployment nginx-deployment
```

2. Verify the Pods are deleted

```sh
kubectl get pods -l app=nginx           
```

The output is similar to this:

```sh
No resources found in default namespace.
```
