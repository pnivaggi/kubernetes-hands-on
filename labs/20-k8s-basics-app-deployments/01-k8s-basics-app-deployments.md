# Application Deployments Lab

- [Application Deployments Lab](#application-deployments-lab)
  - [Objectives](#objectives)
  - [Kubernetes Deployments](#kubernetes-deployments)
  - [Creating an application Deployment](#creating-an-application-deployment)
  - [Inspecting the application Pods](#inspecting-the-application-pods)
  - [Updating a Deployment](#updating-a-deployment)
  - [Scaling a Deployment](#scaling-a-deployment)
  - [Deleting a Deployment](#deleting-a-deployment)

---

## Objectives

- Learn about application Deployments.
- Deploy your first app on Kubernetes with kubectl.
- Update and Scale Deployments.

---

## Kubernetes Deployments

The Deployment instructs Kubernetes how to create and update instances of your application. Once you've created a Deployment, the Kubernetes control plane schedules the application instances included in that Deployment to run on individual Nodes in the cluster.  

Once the application instances are created, a Kubernetes Deployment Controller continuously monitors those instances. If the Node hosting an instance goes down or is deleted, the Deployment controller replaces the instance with an instance on another Node in the cluster. 

> This provides a self-healing mechanism to address machine failure or maintenance.  

In a pre-orchestration world, installation scripts would often be used to start applications, but they did not allow recovery from machine failure. By both creating your application instances and keeping them running across Nodes, Kubernetes Deployments provide a fundamentally different approach to application management.  

---

## Creating an application Deployment

You can create and manage a Deployment by using the Kubernetes command line interface, Kubectl. Kubectl uses the Kubernetes API to interact with the cluster. In this module, you'll learn the most common Kubectl commands needed to create Deployments that run your applications on a Kubernetes cluster.  

When you create a Deployment, you'll need to specify the container image for your application and the number of replicas that you want to run. You can change that information later by updating your Deployment.  

You can describe a Deployment in a YAML file. For your first Deployment, we will use this YAML file which describes a Deployment that runs the NGINX application packaged in Docker image: [01-nginx-deployment.yml](01-nginx-deployment.yml). From the YAML manifest we can see that ngnix release 1.14.2 is used for the container image and 2 replicas are deployed.  

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

4. Create a POD_NAME variable for convenience:

Using grep/head/awd tools on kubectl stdout output:

```sh
export POD_NAME=$(kubectl get pods -l app=nginx | grep Running | head -1 | awk '{print $1}')
```

Using go-template as kubectl output option:

```sh
export POD_NAME=$(kubectl get pods -o go-template='{{(index .items 0).metadata.name}}{{"\n"}}')
```

Using jq to filter json output:

```sh
export POD_NAME=$(kubectl get pods -o json | jq -r '. | .items[0].metadata.name')
```

5. Display information about a Pod:

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

---

## Inspecting the application Pods 

1. Get access to the Pod:

```sh 
kubectl exec -ti $POD_NAME -- /bin/bash
```

The output is similar to this:

```sh
root@nginx-deployment-559d658b74-g8g45:/#
```

2. Try some shell commands on the Pod:

Check the Pod linux release:

```sh
cat /etc/*release
```

The output is similar to this:

```sh
PRETTY_NAME="Debian GNU/Linux 10 (buster)"
NAME="Debian GNU/Linux"
VERSION_ID="10"
VERSION="10 (buster)"
VERSION_CODENAME=buster
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
```

3. Update the Pod packages:

```sh
apt update
```

The output is similar to this:

```sh
Get:1 http://deb.debian.org/debian buster InRelease [122 kB]
Get:2 http://deb.debian.org/debian buster-updates InRelease [51.9 kB]  
Get:3 http://security.debian.org/debian-security buster/updates InRelease [65.4 kB]
Get:4 http://deb.debian.org/debian buster/main amd64 Packages [7906 kB]
Get:5 http://deb.debian.org/debian buster-updates/main amd64 Packages [8792 B]
Get:6 http://security.debian.org/debian-security buster/updates/main amd64 Packages [313 kB]
Fetched 8467 kB in 2s (5025 kB/s)                       
Reading package lists... Done
Building dependency tree       
Reading state information... Done
26 packages can be upgraded. Run 'apt list --upgradable' to see them.
```

4. Install pacakges on the Pod: 

```sh
apt-get install curl iproute2 -y
```

5. Check IP address of the Pod:

```sh
ip add
```

The output is similar to this:

```sh
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8981 qdisc noqueue state UP group default 
    link/ether ea:36:df:63:2a:d6 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 100.106.133.206/32 brd 100.106.133.206 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::e836:dfff:fe63:2ad6/64 scope link 
       valid_lft forever preferred_lft forever
```

6. Check Nginx is working:

Access the default home page of Ngnix app:

```sh
curl 127.0.0.1
```

The output is similar to this:

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

7. Exit the Pod:

```sh
exit
```

8. Check the IP addresses are matching

Compare the IP address seen from the Pod at Step 5. above with the IP address given by kubectl command:

```sh
kubectl get pods -o wide
```

The output is similar to this:

```sh
NAME                                READY   STATUS    RESTARTS   AGE   IP                NODE                           NOMINATED NODE   READINESS GATES
nginx-deployment-559d658b74-g8g45   1/1     Running   0          17h   100.106.133.207   ip-172-20-58-60.ec2.internal   <none>           <none>
nginx-deployment-559d658b74-nb7vj   1/1     Running   0          17h   100.106.133.206   ip-172-20-58-60.ec2.internal   <none>           <none>
nginx-deployment-559d658b74-vw8vm   1/1     Running   0          17h   100.115.37.76     ip-172-20-50-50.ec2.internal   <none>           <none>
nginx-deployment-559d658b74-zb8q6   1/1     Running   0          17h   100.115.37.77     ip-172-20-50-50.ec2.internal   <none>           <none>
```

---

## Updating a Deployment

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

---

## Scaling a Deployment 

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
```

---

## Deleting a Deployment

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
