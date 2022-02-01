# Pod Networking Lab

- [Pod Networking Lab](#pod-networking-lab)
  - [Objectives](#objectives)
  - [Pod Networking Overview](#pod-networking-overview)
  - [Understanding Nodes Networking](#understanding-nodes-networking)
  - [Checking Pods Networking](#checking-pods-networking)
  - [Connectivity between Pods](#connectivity-between-pods)
  - [Connectivity to external networks](#connectivity-to-external-networks)

---

## Objectives

- Learn about Pod networking.
- Understand host and Pod interface pairs.
- Deploy your first Pods on Kubernetes with kubectl.
- Label Nodes to select specific node for Pods.
- Execute command on Pods to understand network connectivity.

---

## Pod Networking Overview

The Pod is the smallest unit in Kubernetes, so it is essential to understand Kubernetes networking in the context of communication between Pods. Kubernetes approach to networking relies on some basic principles:

- Any Pod can communicate with any other Pod without the use of network address translation (NAT). To facilitate this, Kubernetes assigns each Pod an IP address that is routable within the cluster.
- A node can communicate with a Pod without the use of NAT.
- A Pod's awareness of its address is the same as how other resources see the address. The host's address doesn't mask it.

These principles give a unique and first-class identity to every Pod in the cluster. Because of this, the networking model is straightforward and does not need to include port mapping for the running container workloads. 

---

## Understanding Nodes Networking

Kubernetes runs your workload by placing containers into Pods to run on Nodes. A node may be a virtual or physical machine, depending on the cluster. Each node is managed by the control plane and contains the services necessary to run Pods. In our labs, Nodes are setup on AWS cloud as EC2 instances. The cluster includes 1 node running the control plane and 2 nodes supporting the workloads (worker nodes). Let's understand the IP environment for these nodes for you cluster:

1. Get the Nodes list

```bash
kubectl get nodes -o wide
```

- The output is similar to this:

```
NAME                           STATUS   ROLES                  AGE     VERSION   INTERNAL-IP    EXTERNAL-IP    OS-IMAGE             KERNEL-VERSION    CONTAINER-RUNTIME
ip-172-20-50-50.ec2.internal   Ready    node                   4d13h   v1.22.5   172.20.50.50   3.90.60.2      Ubuntu 20.04.3 LTS   5.11.0-1027-aws   containerd://1.4.12
ip-172-20-58-60.ec2.internal   Ready    node                   4d13h   v1.22.5   172.20.58.60   54.81.74.225   Ubuntu 20.04.3 LTS   5.11.0-1027-aws   containerd://1.4.12
ip-172-20-62-7.ec2.internal    Ready    control-plane,master   4d13h   v1.22.5   172.20.62.7    3.81.70.129    Ubuntu 20.04.3 LTS   5.11.0-1027-aws   containerd://1.4.12
```

The `INTERNAL-IP` address corresponds to the node IP address when the `EXTERNAL-IP` address is the corresponding NATed address which is exposed by AWS Internet Gateway to the rest of the world. 

You can also customize the information displayed by kubectl by filtering the json resources from the output with custom columns. For example, if you are interested in the nodes name, role and internal IP address you can execute:

```bash
kubectl get nodes -o wide -o=custom-columns="NAME:metadata.name","ROLE:metadata.labels.kubernetes\.io/role",'InternalIP:status.addresses[?(@.type == "InternalIP")].address'
```

- The output is similar to this:

```bash
NAME                           ROLE     InternalIP
ip-172-20-50-50.ec2.internal   node     172.20.50.50
ip-172-20-58-60.ec2.internal   node     172.20.58.60
ip-172-20-62-7.ec2.internal    master   172.20.62.7
```

1. Export node names and IP addresses

```bash
export node1="$(kubectl get node --selector='!node-role.kubernetes.io/master' -o go-template='{{(index .items 0).metadata.name}}')"
```

```bash
export node2="$(kubectl get node --selector='!node-role.kubernetes.io/master' -o go-template='{{(index .items 1).metadata.name}}')"
```

```bash
export node1_ip=$(kubectl get node --selector='!node-role.kubernetes.io/master' -o go-template='{{(index (index .items 0).status.addresses 1).address}}{{"\n"}}')
```

```bash
export node2_ip=$(kubectl get node --selector='!node-role.kubernetes.io/master' -o go-template='{{(index (index .items 1).status.addresses 1).address}}{{"\n"}}')
```

2. Describe in details one of the nodes

```bash
kubectl describe node $node1
```

- The output is similar to this:
  
```bash
Name:               ip-172-20-50-50.ec2.internal
Roles:              node
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=t3.medium
                    beta.kubernetes.io/os=linux
                    failure-domain.beta.kubernetes.io/region=us-east-1
                    failure-domain.beta.kubernetes.io/zone=us-east-1c
                    kops.k8s.io/instancegroup=nodes-us-east-1c
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=ip-172-20-50-50.ec2.internal
                    kubernetes.io/os=linux
                    kubernetes.io/role=node
                    node-role.kubernetes.io/node=
                    node.kubernetes.io/instance-type=t3.medium
                    topology.ebs.csi.aws.com/zone=us-east-1c
                    topology.kubernetes.io/region=us-east-1
                    topology.kubernetes.io/zone=us-east-1c
Annotations:        csi.volume.kubernetes.io/nodeid: {"ebs.csi.aws.com":"i-0380462c4c37c6b7d"}
                    node.alpha.kubernetes.io/ttl: 0
                    projectcalico.org/IPv4Address: 172.20.50.50/19
                    projectcalico.org/IPv4IPIPTunnelAddr: 100.115.37.64
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Thu, 27 Jan 2022 18:18:39 +0100
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  ip-172-20-50-50.ec2.internal
  AcquireTime:     <unset>
  RenewTime:       Tue, 01 Feb 2022 11:29:21 +0100
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Thu, 27 Jan 2022 18:18:53 +0100   Thu, 27 Jan 2022 18:18:53 +0100   CalicoIsUp                   Calico is running on this node
  MemoryPressure       False   Tue, 01 Feb 2022 11:26:08 +0100   Thu, 27 Jan 2022 18:18:39 +0100   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Tue, 01 Feb 2022 11:26:08 +0100   Thu, 27 Jan 2022 18:18:39 +0100   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Tue, 01 Feb 2022 11:26:08 +0100   Thu, 27 Jan 2022 18:18:39 +0100   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Tue, 01 Feb 2022 11:26:08 +0100   Thu, 27 Jan 2022 18:18:59 +0100   KubeletReady                 kubelet is posting ready status. AppArmor enabled
Addresses:
  InternalIP:   172.20.50.50
  ExternalIP:   3.90.60.2
  Hostname:     ip-172-20-50-50.ec2.internal
  InternalDNS:  ip-172-20-50-50.ec2.internal
  ExternalDNS:  ec2-3-90-60-2.compute-1.amazonaws.com
Capacity:
  cpu:                2
  ephemeral-storage:  130045936Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3959672Ki
  pods:               110
Allocatable:
  cpu:                2
  ephemeral-storage:  119850334420
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3857272Ki
  pods:               110
System Info:
  Machine ID:                 ec278960174dcf7110f059fac8c0956f
  System UUID:                ec278960-174d-cf71-10f0-59fac8c0956f
  Boot ID:                    a80ede58-e007-4488-bc61-7722900366a4
  Kernel Version:             5.11.0-1027-aws
  OS Image:                   Ubuntu 20.04.3 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.4.12
  Kubelet Version:            v1.22.5
  Kube-Proxy Version:         v1.22.5
PodCIDR:                      100.96.1.0/24
PodCIDRs:                     100.96.1.0/24
ProviderID:                   aws:///us-east-1c/i-0380462c4c37c6b7d
Non-terminated Pods:          (6 in total)
  Namespace                   Name                                       CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                       ------------  ----------  ---------------  -------------  ---
  kube-system                 calico-node-58sl6                          100m (5%)     0 (0%)      0 (0%)           0 (0%)         4d17h
  kube-system                 coredns-5dc785954d-rxd5p                   100m (5%)     0 (0%)      70Mi (1%)        170Mi (4%)     4d17h
  kube-system                 coredns-5dc785954d-wrjpm                   100m (5%)     0 (0%)      70Mi (1%)        170Mi (4%)     4d17h
  kube-system                 coredns-autoscaler-84d4cfd89c-x67r5        20m (1%)      0 (0%)      10Mi (0%)        0 (0%)         4d17h
  kube-system                 ebs-csi-node-dl7k6                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         4d17h
  kube-system                 kube-proxy-ip-172-20-50-50.ec2.internal    100m (5%)     0 (0%)      0 (0%)           0 (0%)         4d17h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                420m (21%)  0 (0%)
  memory             150Mi (3%)  340Mi (9%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
Events:              <none>
```

We can see under `Labels` section that the node resource is labeled and we will use it later to deploy Pod on specific node.  

Please note the `Annotations` section used by some plugins to provide metada like for the network CNI Calico `projectcalico.org/IPv4Address` and `projectcalico.org/IPv4IPIPTunnelAddr`. Annotations can be used to configure extended resources not part of the Kubernetes core API.  

Check the `PodCIDR` used for the Node.  

Finally you can check the Pod running on the Node under the `Non-terminated Pods` section.  

3. Check the interfaces on the node

- Get ssh access to the node1

```bash
ssh ubuntu@$node1_ip
```

Execute ip address command and check details for `ens5` and `tunl0`:

- The output is similar to this:

```bash
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 0e:f7:95:2f:58:59 brd ff:ff:ff:ff:ff:ff
    altname enp0s5
    inet 172.20.50.50/19 brd 172.20.63.255 scope global dynamic ens5
       valid_lft 2780sec preferred_lft 2780sec
    inet6 fe80::cf7:95ff:fe2f:5859/64 scope link 
       valid_lft forever preferred_lft forever
3: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 8981 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
    inet 100.115.37.64/32 scope global tunl0
       valid_lft forever preferred_lft forever
6: cali2a727d7f97c@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8981 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-44553947-914e-14f6-e5c1-0ec57a805e0c
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
7: cali01acfe2f52d@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8981 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-1d6000f7-6561-b18d-14d4-f683738b2c6e
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
8: cali3f52c3fc33f@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8981 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-1cfde65b-cd7d-072d-a3a3-12e4855352b9
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
9: calied4dc1027ee@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8981 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-f74f5a26-87ca-8ef0-3b40-9fcdfa5d7d71
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
```

`ens5` corresponds to the node ethernet interface and `tunl0` corresponds to the interface tunnel used by the overlay network to interconnect the different cluster nodes if not on the same subnet and allow inter node Pod communication.  

Note the other interfaces with the name starting with `cali`. These correspond to veth pairs where one end of the pair `@if4` is the Pod interface in the Pod network namespace and the other end `cali...` is the other end of the pair in the host network namespace.  

4. Check the routes installed on the node

```bash
ip route
```

- The output is similar to this:

```bash
default via 172.20.32.1 dev ens5 proto dhcp src 172.20.50.50 metric 100 
100.104.132.0/26 via 172.20.62.7 dev ens5 proto bird 
100.106.133.192/26 via 172.20.58.60 dev ens5 proto bird 
blackhole 100.115.37.64/26 proto bird 
100.115.37.65 dev cali2a727d7f97c scope link 
100.115.37.66 dev cali01acfe2f52d scope link 
100.115.37.67 dev cali3f52c3fc33f scope link 
100.115.37.68 dev calied4dc1027ee scope link 
172.20.32.0/19 dev ens5 proto kernel scope link src 172.20.50.50 
172.20.32.1 dev ens5 proto dhcp scope link src 172.20.50.50 metric 100
```

Note the default route via the host ethernet interface `ens5`. More interesting are the routes related to:

- `bird`: we use Calico CNI in our labs, the reachability of the Pods deployed on the remote nodes are assured by these routes learned from BGP (bird is a linux application for BGP routing), Calico CNI use BGP to advertise/learn routes to/from the other nodes. All the nodes are deployed on the same AWS subnet therefore the next hop for the BGP prefixes correspond to the IP address of the remote nodes.
- `cali...`: Calico CNI configure a host route via veth pair to give reachability to the Pod locally
- `blakhole`: correspond to the BGP aggregate route for the locally deployed Pods that bird will announce to the other nodes of the cluster

5. Exit the shell

```bash
exit
```

## Checking Pods Networking

We will deploy 2 Pods (named bb1 and bb2) running the `busybox` image. Busybox image includes tools to check IP connectivity like wget, nc, ping, traceroute... In addition we will deploy another Pods named web running a `nginx` image. It will be useful to test HTTP client/server kind of communications.  We want also to deploy Pods on different nodes to test the inter-node communication, therefore bb1 is deployed on node1 and bb2 on node2.

1. Configure label on nodes

We use a label called alias to segregate between the nodes. Make sure you execute the command from your laptop or provided host but not from the node.

```bash
kubectl label nodes $node1 alias=node1
```

```bash
kubectl label nodes $node2 alias=node2
```

2. Check the label alias on the nodes

```bash
kubectl get nodes -o wide -o=custom-columns="NAME:metadata.name","ROLE:metadata.labels.kubernetes\.io/role",'ALIAS:metadata.labels.alias'
```

- The output is similar to this:

```bash
NAME                           ROLE     ALIAS
ip-172-20-50-50.ec2.internal   node     node1
ip-172-20-58-60.ec2.internal   node     node2
ip-172-20-62-7.ec2.internal    master   <none>
```

3. Deploy Pod on node1 and node2

- Deploy bb1 Pod with the busybox image on node1

```bash
kubectl run -i --tty bb1 --image=busybox --overrides='{"spec": { "nodeSelector": {"alias": "node1"}}}'
```

- The output is similar to this:

```bash
If you don't see a command prompt, try pressing enter.
/ # 
```

- Exit the container

```bash
exit
```

- Deploy bb2 Pod with the busybox image on node2

```bash
kubectl run -i --tty bb2 --image=busybox --overrides='{"spec": { "nodeSelector": {"alias": "node2"}}}'
```

- The output is similar to this:

```bash
If you don't see a command prompt, try pressing enter.
/ # 
```

- Exit the container

```bash
exit
```

- Check the Pods are deployed on the right nodes

```bash
kubectl get pods bb1 bb2 -n default -o wide
```

- The output is similar to this:
  
```bash
NAME   READY   STATUS    RESTARTS       AGE    IP                NODE                           NOMINATED NODE   READINESS GATES
bb1    1/1     Running   1 (103m ago)   103m   100.115.37.81     ip-172-20-50-50.ec2.internal   <none>           <none>
bb2    1/1     Running   1 (100m ago)   100m   100.106.133.221   ip-172-20-58-60.ec2.internal   <none>           <none>
```

```bash
echo $node1
```

- The output is similar to this:

```bash
ip-172-20-50-50.ec2.internal
```

and it concludes bb1 is well deployed on node1

```bash
echo $node2
```

- The output is similar to this:

```bash
ip-172-20-58-60.ec2.internal
```

and it concludes bb2 is well deployed on node2

4. Check node interface/route modifications

- Get ssh access to node1 and inspect routes and interfaces

```bash
ssh ubuntu@$node1_ip
```

```bash
ip route
```

- The output is similar to this:

```bash
default via 172.20.32.1 dev ens5 proto dhcp src 172.20.50.50 metric 100 
100.104.132.0/26 via 172.20.62.7 dev ens5 proto bird 
100.106.133.192/26 via 172.20.58.60 dev ens5 proto bird 
blackhole 100.115.37.64/26 proto bird 
100.115.37.65 dev cali2a727d7f97c scope link 
100.115.37.66 dev cali01acfe2f52d scope link 
100.115.37.67 dev cali3f52c3fc33f scope link 
100.115.37.68 dev calied4dc1027ee scope link 
100.115.37.81 dev cali98fd8882b62 scope link 
172.20.32.0/19 dev ens5 proto kernel scope link src 172.20.50.50 
172.20.32.1 dev ens5 proto dhcp scope link src 172.20.50.50 metric 100
```

- Identify the bb1 Pod ip address in the host route. In our case it is ```100.115.37.81 dev cali98fd8882b62 scope link```.

It means that Calico CNI has created a host route for the new Pod bb1. 

```bash
ip addr
```

```bash
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 0e:f7:95:2f:58:59 brd ff:ff:ff:ff:ff:ff
    altname enp0s5
    inet 172.20.50.50/19 brd 172.20.63.255 scope global dynamic ens5
       valid_lft 1905sec preferred_lft 1905sec
    inet6 fe80::cf7:95ff:fe2f:5859/64 scope link 
       valid_lft forever preferred_lft forever
3: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 8981 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
    inet 100.115.37.64/32 scope global tunl0
       valid_lft forever preferred_lft forever
6: cali2a727d7f97c@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8981 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-44553947-914e-14f6-e5c1-0ec57a805e0c
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
7: cali01acfe2f52d@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8981 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-1d6000f7-6561-b18d-14d4-f683738b2c6e
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
8: cali3f52c3fc33f@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8981 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-1cfde65b-cd7d-072d-a3a3-12e4855352b9
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
9: calied4dc1027ee@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8981 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-f74f5a26-87ca-8ef0-3b40-9fcdfa5d7d71
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
22: cali98fd8882b62@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8981 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netns cni-e96ac8dc-a9ae-d6f8-dd4c-c00ecbbb7c70
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
```

- Identify Calico CNI has also created a veth pair to link the Pod interface to the host network namespace. In our case it is interface (22:) named ```cali98fd8882b62@if4```.

From the interface naming convention, we can figure out the veth pair is connected to interface (4:) on the Pod side. Let's check that on the Pod itself.

5. Check Pod IP address and connectivity

- Export the IP address of Pod bb1 

```bash
export bb1_ip=$(kubectl get pod bb1 -o go-template --template '{{.status.podIP}}')
```

- Visualize that IP address allocated by Kubernetes

```bash
echo $bb1_ip
```

- Check it is the same address as the one seen by the Pod itself

```bash
kubectl exec -i -t bb1 -- ip add 
```

- The output is similar to this:

```bash
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if22: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 8981 qdisc noqueue 
    link/ether 52:77:84:c6:5a:e0 brd ff:ff:ff:ff:ff:ff
    inet 100.115.37.81/32 brd 100.115.37.81 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::5077:84ff:fec6:5ae0/64 scope link 
       valid_lft forever preferred_lft forever
```

In our case, IP address seen by the Pod bb1 is `100.115.37.81/32` and it is the same as the one given by kubectl get pods command.

We notice also the interface on the Pod is the fourth one (4:) as indicated when inputing ip addr command on node1, that interface named `eth0@if22` is also connectected the interface (22:) in the host network namespace as indicated by `@if22`.

## Connectivity between Pods

- Execute traceroute command from Pod bb2 (on node2) to Pod bb1 (on node1)

```bash
kubectl exec -i -t bb2 -- traceroute $bb1_ip
```

- The output is similar to this:

```bash
traceroute to 100.115.37.81 (100.115.37.81), 30 hops max, 46 byte packets
 1  ip-172-20-58-60.ec2.internal (172.20.58.60)  0.006 ms  0.006 ms  0.005 ms
 2  ip-172-20-50-50.ec2.internal (172.20.50.50)  0.205 ms  0.219 ms  0.166 ms
 3  100.115.37.81 (100.115.37.81)  0.241 ms  0.809 ms  0.222 ms
 ```

 We notice the path includes node2 (in our case ip-172-20-58-60.ec2.internal) which is the node where Pod bb2 sits, then node1 (in our case ip-172-20-50-50.ec2.internal) which is the node where Pod bb1 sits and finally Pod bb1 (in our case 100.115.37.81).  

## Connectivity to external networks

- Execute a ping to a well known address

```bash
kubectl exec -i -t bb1 -- ping -c 5 www.google.com
```

- The output is similar to this:

```bash
PING www.google.com (142.251.45.100): 56 data bytes
64 bytes from 142.251.45.100: seq=0 ttl=110 time=2.466 ms
64 bytes from 142.251.45.100: seq=1 ttl=110 time=1.854 ms
64 bytes from 142.251.45.100: seq=2 ttl=110 time=1.590 ms
64 bytes from 142.251.45.100: seq=3 ttl=110 time=1.531 ms
64 bytes from 142.251.45.100: seq=4 ttl=110 time=1.571 ms

--- www.google.com ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 1.531/1.802/2.466 ms
```

Note that in addition to external connectivity, the Pod can also resolve DNS name.

- Check the IP address of the Pod seen by the external networks

```bash
kubectl exec -i -t bb1 -- wget -qO- https://api.ipify.org
```

- The output is similar to this:

```bash
wget: note: TLS certificate validation not implemented
3.90.60.2
```

We notice the Pod bb1 IP address seen by the external world is the same has node1 external IP address. Pods have connectivity to outside by default and the traffic is source NATed with the IP address of the node hosting the Pod.