# IP Pools and Services Advertisement

- [IP Pools and Services Advertisement](#ip-pools-and-services-advertisement)
  - [Objectives](#objectives)
  - [Pod and Service IP Pools advertisement](#pod-and-service-ip-pools-advertisement)
  - [Pod external routable IP Pool](#pod-external-routable-ip-pool)
  - [BGP peering and IP Pool advertisement](#bgp-peering-and-ip-pool-advertisement)

## Objectives

## Pod and Service IP Pools advertisement

There are two address ranges that Kubernetes is normally configured with that are worth understanding:  

- The cluster pod CIDR is the range of IP addresses Kubernetes is expecting to be assigned to pods in   the cluster.
- The services CIDR is the range of IP addresses that are used for the Cluster IPs of Kubernetes Sevices (the virtual IP that corresponds to each Kubernetes Service).

The lab uses Calico CNI which allows advanced feature like using BGP to advertise IP prefixes related to Pod or services IP addresses range.
One use of Calico IP Pools is to distinguish between different ranges of addresses with different routability scopes.

## Pod external routable IP Pool

1. Check the configured IP Pools

```bash
calicoctl get ippools --allow-version-mismatch
```

The output is similar to:

```bash
NAME                  CIDR            SELECTOR   
default-ipv4-ippool   100.96.0.0/11   all()  
```

In this cluster Calico has been configured to allocate IP addresses for pods from the 100.96.0.0/11 CIDR.

2. Create externally routable IP Pool

```bash
calicoctl apply -f 02-externalIpPool.yml --allow-version-mismatch
```

The output is similar to:

```bash
Successfully applied 1 'IPPool' resource(s)
```

- Check the IPPool configured in the cluster

```bash
calicoctl get ippools --allow-version-mismatch
```

The output is similar to:

```bash
NAME                  CIDR             SELECTOR   
default-ipv4-ippool   100.96.0.0/11    all()      
external-pool         198.19.24.0/21   !all() 
```

3. Create external namespace

```bash
kubectl apply -f 03-externalNamespace.yml
```

The output is similar to:

```bash
namespace/external-ns created
```

4. Deploy nginx application

```bash
kubectl apply -f 04-nginx-deployment-ns.yml
```

The output is similar to:

```bash
deployment.apps/nginx created
```

5. Check IP address allocated

```bash
kubectl get pods -n external-ns -o wide
```

The output is similar to:

```bash
NAME                     READY   STATUS    RESTARTS   AGE   IP              NODE                           NOMINATED NODE   READINESS GATES
nginx-76dd8577bc-8lj92   1/1     Running   0          13h   198.19.24.184   ip-172-20-58-60.ec2.internal   <none>           <none>
```

Notice the ngnix pod has been allocated an address (198.19.24.184) belonging to the external-pool.

## BGP peering and IP Pool advertisement

1. Configure BGP to peer with Bird VM

```bash
calicoctl apply -f 01.globalBgpPeer.yml --allow-version-mismatch
```

The output is similar to:

```bash
Successfully applied 1 'BGPPeer' resource(s)
```

2. Check the BGP connection status

- Connect to node1

```bash
export node1_ip=$(kubectl get node --selector='!node-role.kubernetes.io/master' -o go-template='{{(index (index .items 0).status.addresses 1).address}}{{"\n"}}')
```

```bash
ssh ubuntu@$node1_ip
```

- Check BGP status on node1
  
```bash
sudo calicoctl node status
```

The output is similar to:

```bash
Calico process is running.

IPv4 BGP status
+---------------+-------------------+-------+------------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |   SINCE    |    INFO     |
+---------------+-------------------+-------+------------+-------------+
| 172.20.62.7   | node-to-node mesh | up    | 2022-01-27 | Established |
| 172.20.58.60  | node-to-node mesh | up    | 2022-01-27 | Established |
| 172.20.34.231 | global            | up    | 16:57:11   | Established |
+---------------+-------------------+-------+------------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

- Connect to Bird VM

```bash
sudo birdc show protocol
```

The output is similar to:

```bash
BIRD 2.0.7 ready.
Name       Proto      Table      State  Since         Info
device1    Device     ---        up     16:57:09.492  
direct1    Direct     ---        up     16:57:09.492  
kernel1    Kernel     master4    up     16:57:09.492  
kernel2    Kernel     master6    up     16:57:09.492  
static1    Static     master4    up     16:57:09.492  
node1      BGP        ---        up     06:03:30.732  Established   
node2      BGP        ---        up     06:03:30.732  Established 
```

Notice the two sessions established with node1 and node2.

3. Check the IP routes on node1

- connect to node1

```bash
ssh ubuntu@$node1_ip
```

- verify route on node1
  
```bash
ip route
```

The output is similar to:

```bash
default via 172.20.32.1 dev ens5 proto dhcp src 172.20.58.60 metric 100 
1.1.1.1 via 172.20.34.231 dev ens5 proto bird 
10.10.10.10 via 172.20.34.231 dev ens5 proto bird 
100.104.132.0/26 via 172.20.62.7 dev ens5 proto bird 
blackhole 100.106.133.192/26 proto bird 
100.106.133.193 dev calic5dc3ed6bc4 scope link 
100.106.133.221 dev cali9270933bb0b scope link 
100.106.133.234 dev cali6e223e9f925 scope link 
100.115.37.64/26 via 172.20.50.50 dev ens5 proto bird 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
172.20.32.0/19 dev ens5 proto kernel scope link src 172.20.58.60 
172.20.32.1 dev ens5 proto dhcp scope link src 172.20.58.60 metric 100 
198.19.24.184 dev calie1b64f5d824 scope link 
blackhole 198.19.24.184/29 proto bird
```

Notice the routes learned from BGP bird process (advertised by the Bird VM) and the new routes related to ngnix deployment:
- ```198.19.24.184 dev calie1b64f5d824 scope link```: host route to reach the ngnix Pod
- ```blackhole 198.19.24.184/29 proto bird```: prefix advertised by the node1 to external BGP peers

4. Check the IP routes on Bird VM

```bash
$ sudo birdc show route
BIRD 2.0.7 ready.
Table master4:
198.19.24.184/29     unicast [node1 2022-02-10] * (100) [AS64512i]
        via 172.20.50.50 on eth0
                     unicast [node2 2022-02-10] (100) [AS64512i]
        via 172.20.58.60 on eth0
1.1.1.1/32           unicast [direct1 2022-02-09] * (240)
        dev dummy1
10.10.10.10/32       blackhole [static1 2022-02-09] * (200)
100.115.37.64/26     unicast [node1 2022-02-10] * (100) [AS64512i]
        via 172.20.50.50 on eth0
                     unicast [node2 2022-02-10] (100) [AS64512i]
        via 172.20.58.60 on eth0
100.106.133.192/26   unicast [node1 2022-02-10] * (100) [AS64512i]
        via 172.20.50.50 on eth0
                     unicast [node2 2022-02-10] (100) [AS64512i]
        via 172.20.58.60 on eth0
172.20.32.0/19       unicast [direct1 2022-02-10] * (240)
        dev eth0
100.104.132.0/26     unicast [node1 2022-02-10] * (100) [AS64512i]
        via 172.20.50.50 on eth0
                     unicast [node2 2022-02-10] (100) [AS64512i]
        via 172.20.58.60 on eth0
```

Notice that 198.19.24.184/29 prefix is advertised from node1 and node2
