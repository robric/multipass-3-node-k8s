# Looking under the hood: clusterIP, nodeport and metallb

This page is:
- a quick copy-paste source for rapidly bringing up test scenarios.
- an educational source so I can quickly share how things work under the the hood when interfacing with diverse people... Indeed, few people actually understand the kubernetes networking logic.
- a personal cheat sheet for k8s for fast refresh.

## Prerequisite

I just start with an ubuntu server with:
- multipass installed (by default on ubuntu)
- a keypair in .ssh/ (id_rsa.pub) so we can simply access VMs

## VM and Cluster deployment 

The testings are based on a deployment of 4 VMs via multipass and additional networking.
- 1 Cluster made up of 3 vms (k3s).
- 1 external VM to run testings to reach the cluster in load balancer services.

The below diagram captures the main of characteristics of the lab topology.

```
Networks:
- native multipass on bridge mpqemubr0 10.65.94.0/24
- external network on vlan 100 (10.123.123.0/24). This is for the testings of
Load Balancers services.

Kubernetes Cluster VM:
- vm1: master/worker - External IP address = 10.123.123.1/24
- vm2: worker - External IP address = 10.123.123.2/24
- vm3: worker - External IP address = 10.123.123.3/24

External VM: 
- vm-ext: External IP address = 10.123.123.4/24

            +----------------------------------------------+    
            |                   Cluster                    |    
            |                                              |     
            |    MASTER                                    |
            |      +                                       |   External VM
            |     worker          worker        worker     |         
            | +------------+ +------------+ +------------+ | +------------+  
            | |     vm1    | |    vm2     | |    vm3     | | |     vm-ext |   
            | +----ens3----+ +---ens3-----+ +---ens3-----+ | +----ens3----+  
            |      :  |.1/24      :  |.2/24      :  |.3/24 |       :  |.4/24   
            +------:--|-----------:--|-----------:--|------+       :  |
                   :  |           :  |           :  |              :  |         
                   :  |           :  |           :  |              :  |    
                   :  |           :  |           :  |              :  |    
                   :  +=========vlan +100 (external)+=================+    
                   :              10.123.123.0/24:  |              :
                   :              :              :  |              :   
                   :              :              :  |              : 
                   :              :              :  |              : 
                   :              :              :  |              :                      
             ------+------ native +vlan(internal)+-----------------+               
                        :         10.65.94.0/24     |   
                        :                           |
                        :                           |
                 [mpqemubr0]                  [mpqemubr0.100]                         
                 10.65.94.1/24               10.123.123.254/24                          
                     host                          host
```
To deploy this, just run the following command. It will deploy the network plumbing and vms.
```
curl -sSL https://raw.githubusercontent.com/robric/multipass-3-node-k8s/main/source/deploy.sh | sh
```
You'll get the 4 VMs with kubernetes running (k3s inside)
```console
root@fiveg-host-24-node4:~# multipass list 
Name                    State             IPv4             Image
vm-ext                  Running           10.65.94.56      Ubuntu 22.04 LTS
                                          10.123.123.4
vm1                     Running           10.65.94.121     Ubuntu 22.04 LTS
                                          10.42.0.0
                                          10.42.0.1
                                          10.123.123.1
vm2                     Running           10.65.94.22      Ubuntu 22.04 LTS
                                          10.42.1.0
                                          10.123.123.2
vm3                     Running           10.65.94.156     Ubuntu 22.04 LTS
                                          10.42.2.0
                                          10.123.123.3
root@fiveg-host-24-node4:~# multipass shell vm1
[...]
Last login: Tue Jun 11 01:57:04 2024 from 10.65.94.1
ubuntu@vm1:~$ kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
vm1    Ready    control-plane,master   22h   v1.29.5+k3s1
vm3    Ready    <none>                 22h   v1.29.5+k3s1
vm2    Ready    <none>                 22h   v1.29.5+k3s1
ubuntu@vm1:~$ 
```

##  K8s basic service deployment and routing analysis

### Bootstraping

Let's start with the creation of a test pod netshoot (https://github.com/nicolaka/netshoot). This is a great troubleshooting container for exploring networking.
```
kubectl run test-pod --image=nicolaka/netshoot --command -- sleep infinity
```

Next let's start a with basic service (cluster IP). This will create a multi-node cluster made up of 3VM based on k3s. 

```
kubectl apply -f https://raw.githubusercontent.com/robric/multipass-3-node-k8s/main/source/nginx-svc.yaml
```

This is what we get:

```console
ubuntu@vm1:~$ kubectl get svc -o wide
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE   SELECTOR
kubernetes      ClusterIP   10.43.0.1       <none>        443/TCP   22h   <none>
nginx-service   ClusterIP   10.43.180.238   <none>        80/TCP    22h   app=nginx
ubuntu@vm1:~$ kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP          NODE   NOMINATED NODE   READINESS GATES
nginx-deployment-7c79c4bf97-gk7cj   1/1     Running   0          22h     10.42.0.8   vm1    <none>           <none>
nginx-deployment-7c79c4bf97-4fdl4   1/1     Running   0          22h     10.42.2.2   vm3    <none>           <none>
nginx-deployment-7c79c4bf97-5j5bv   1/1     Running   0          22h     10.42.1.2   vm2    <none>           <none>
test-pod                            1/1     Running   0          4m35s   10.42.1.4   vm2    <none>           <none>
ubuntu@vm1:~$
```
### How does a pod reaches a services ?
Now let's check what happens when a pods reaches a service. Here *test-pod* reaches the *nginx-service*.

```console
ubuntu@vm1:~$ kubectl  exec -it test-pod -- curl http://nginx-service:80/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
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
ubuntu@vm1:~$ 
```
First the pod needs to resolve the URL http://nginx-service:80/ via DNS. So let's check DNS configuration for test-pod.  
```console
ubuntu@vm1:~$ kubectl  exec -it test-pod -- cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local multipass
nameserver 10.43.0.10
options ndots:5
ubuntu@vm1:~$
```
Which is the IP of the -how suprising !- kube-dns service. 
```
ubuntu@vm1:~$ kubectl get svc -A
NAMESPACE      NAME             TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                        AGE
[...]
kube-system    kube-dns         ClusterIP      10.43.0.10      <none>        53/UDP,53/TCP,9153/TCP         22h
```
Hence, before reaching any service, there is a request to the DNS service itself... the svc plumbing is cluster IP just like the nginx-service itself but for DNS trafic (UDP 53). We'll see the deatils of how this works through iptables/NAT later. Ultimately the DNS requests reaches the coredns pod. So let's have a look at it to see how the service name resolution is enforced.
```
.----------.                                                   .---------.   
| test-pod |--(veth)---[kube-dns=10.43.0.10]----------(veth)---| coredns |
'----------'                                                   '---------'  
     |               
     |
+----------------------------------------------------------------                                                                  
|curl http://nginx-service:80/                                  |
|DNS REQUEST to 10.43.0.10: what is the IP for nginx-service ?  | 
|DNS RESPONSE from 10.43.0.10: This is IP address 10.43.180.238 |  
|                                                               |  
+----------------------------------------------------------------

TCPDUMP Traces from pod:

ubuntu@vm1:~$ kubectl  exec -it test-pod -- bash
test-pod:~# tcpdump -vni  eth0
#
# DNS REQUEST      ----   TEST-POD (10.42.1.4) TO DNS SERVICE (10.43.0.1)
#
13:09:28.510565 IP (tos 0x0, ttl 64, id 6023, offset 0, flags [DF], proto UDP (17), length 96)
    10.42.1.4.60609 > 10.43.0.10.53: 44890+ [1au] A? nginx-service.default.svc.cluster.local. (68)
13:09:28.510886 IP (tos 0x0, ttl 64, id 6024, offset 0, flags [DF], proto UDP (17), length 96)
    10.42.1.4.60609 > 10.43.0.10.53: 36406+ [1au] AAAA? nginx-service.default.svc.cluster.local. (68)
[...]
#
# DNS RESPONSE       ----  DNS SERVICE (10.43.0.1) TO TEST-POD (10.42.1.4)
#
13:09:28.512136 IP (tos 0x0, ttl 62, id 51894, offset 0, flags [DF], proto UDP (17), length 151)
    10.43.0.10.53 > 10.42.1.4.60609: 44890*- 1/0/1 nginx-service.default.svc.cluster.local. A 10.43.180.238 (123)
#
# Here we go now pod knows that nginx-service is at 10.43.180.238
#
```

### A glance at coredns  
coredns is started via configmap which has the kube node IPs (here vm1-3). The service name resolution are not stored there since cm this would be highly unpractical: cm are ok for data that permits to start containers with appropriate parameters but not for data that requires to be updated at runtime. 

```console
ubuntu@vm1:~$ kubectl get pods -n kube-system 
NAME                                      READY   STATUS      RESTARTS   AGE
[...]
coredns-6799fbcd5-9ln42                   1/1     Running     0          23h
ubuntu@vm1:~$ kubectl get cm -n kube-system coredns  -o yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        hosts /etc/coredns/NodeHosts {
          ttl 60
          reload 15s
          fallthrough
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
        import /etc/coredns/custom/*.override
    }
    import /etc/coredns/custom/*.server
  NodeHosts: |
    10.65.94.95 vm3
    10.65.94.199 vm2
    10.65.94.238 vm1
ubuntu@vm1:~$ kubectl describe pod -n kube-system coredns-6799fbcd5-9ln42 | grep containerd
    Container ID:  containerd://f5499f73a98b24fbc08a2da797e44f4edd932ce838d3f6b3a5d76e4ce8a0d359

ubuntu@vm1:~$ sudo ctr c info f5499f73a98b24fbc08a2da797e44f4edd932ce838d3f6b3a5d76e4ce8a0d359
[...]
                "destination": "/etc/coredns",
                "type": "bind",
                "source": "/var/lib/kubelet/pods/05cc11a6-6aa4-4da9-b1db-e56cf20d222f/volumes/kubernetes.io~configmap/config-volume",
[...]
                "destination": "/etc/hosts",
                "type": "bind",
                "source": "/var/lib/kubelet/pods/05cc11a6-6aa4-4da9-b1db-e56cf20d222f/etc-hosts",
[...]
                "destination": "/etc/resolv.conf",
                "type": "bind",
                "source": "/var/lib/rancher/k3s/agent/containerd/io.containerd.grpc.v1.cri/sandboxes/0a3004680d3dd773b7ca99e7ec1b85c030bc96b0485f5420c20af9efdf3b3a1b/resolv.conf",
                "options": [
                    "rbind",
                    "rprivate",
                    "ro"
ubuntu@vm1:~$ sudo cat  /var/lib/kubelet/pods/05cc11a6-6aa4-4da9-b1db-e56cf20d222f/volumes/kubernetes.io~configmap/config-volume/Corefile
.:53 {
    errors
    health
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
      pods insecure
      fallthrough in-addr.arpa ip6.arpa
    }
    hosts /etc/coredns/NodeHosts {
      ttl 60
      reload 15s
      fallthrough
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
    import /etc/coredns/custom/*.override
}
import /etc/coredns/custom/*.server
ubuntu@vm1:~$ sudo cat  /var/lib/kubelet/pods/05cc11a6-6aa4-4da9-b1db-e56cf20d222f/etc-hosts
# Kubernetes-managed hosts file.
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
fe00::0 ip6-mcastprefix
fe00::1 ip6-allnodes
fe00::2 ip6-allrouters
10.42.0.2       coredns-6799fbcd5-9ln42
ubuntu@vm1:~$ sudo cat  /etc/resolv.conf
[...]
nameserver 127.0.0.53
options edns0 trust-ad
search multipass
ubuntu@vm1:~$ 
```
Service name resolution works as any kubernetes: the key/values are stored in the kube DB (e.g. etcd) and sent to coredns upon changes.

### Service Routing Details

This section describes the logic for routing kubernetes services which is extensively based on NAT.
In summary:
- after name resolution, the trafic is sent by test-pod to the nginx-service IP address 10.43.180.238
- upon reception on veth interface at host side, DNAT is enforced to translate the service address to any of the pod IP.

This is demonstrated in the below diagram and capture (several attemps for TCPdump were necessary since there is load balancing :-)).
  
```
                                                                 deployment 3 pods
.----------.                                                       .---------.   
| test-pod |--(veth)--[host-vm1]--(routing)----[host-vm1]--(veth)--| nginx-1 | 10.42.0.8
'----------'               ^                 |                     '---------'            
         10.42.1.4         |                 |                     .---------.  
                        [ NAT ]              |-[host-vm3]--(veth)--| nginx-2 | 10.42.2.2  
                           |                 |                     '---------'
                           |                 |                     .---------.  
                           |                 |-[host-vm2]--(veth)--| nginx-3 | 10.42.1.2  
                                                                   '---------'
         iptables: NAT dest = nginx-service(10.43.180.238)

S=10.42.1.4, D=10.43.180.238 ----[NAT]---- S=10.42.1.4, D=10.42.0.8 
(this an example since DEST can be any of the nginx pods since they're loaded balanced)


#
# From test-pod:
#

test-pod:~# tcpdump -vni eth0 "tcp and port 80"

TCP SYN:
    10.42.1.4.49186 > 10.43.180.238.80: Flags [S]

TCP SYN-ACK:
    10.43.180.238.80 > 10.42.1.4.49186: Flags [S.]

#
#From host vm1 (veth interface on dest pod - nginx-1 -): 
#

ubuntu@vm1:~$ sudo tcpdump -vni veth39f5a18d

TCP SYN:
    10.42.1.4.49186 > 10.42.0.8.80: Flags [S]

TCP SYN-ACK:
    10.42.0.8.80 > 10.42.1.4.49186: Flags [S.]
```
*TIP: find out which veth is attached to a pod, since there is no such info kubectl describe.*
```
#
#  if not done: install brctl and net-tools
#

#
# Find your DMAC based on pod IP address that you get from "kubectl get pods -o wide" option.
# Herere 10.42.0.8 for pod nginx-1 = nginx-deployment-7c79c4bf97-gk7cj in vm1
#
ubuntu@vm1:~$ arp -na 
? (10.42.0.8) at 16:9b:6c:12:bd:34 [ether] on cni0
ubuntu@vm1:~$ brctl showmacs  cni0 | grep 16
  2     16:9b:6c:12:bd:34       no                 2.45
#
# First colum is the port number
#
ubuntu@vm1:~$ brctl show cni0
bridge name     bridge id               STP enabled     interfaces
cni0            8000.3e07f3337fda       no              veth06f7413c
                                                        veth39f5a18d <=========== This one is index=2 
```
OK so now let's explore how the NAT plumbing is enforced.
```
#
# First, find out the name of the rules that are involved in translation to service IP = 10.43.180.238 
# Then explore the set of rules that enforce NAT. This is ugly, but that's what iptables is.
# 
ubuntu@vm1:~$ sudo iptables-save | grep  10.43.180.238    
-A KUBE-SERVICES -d 10.43.180.238/32 -p tcp -m comment --comment "default/nginx-service cluster IP" -m tcp --dport 80 -j KUBE-SVC-V2OKYYMBY3REGZOG
-A KUBE-SVC-V2OKYYMBY3REGZOG ! -s 10.42.0.0/16 -d 10.43.180.238/32 -p tcp -m comment --comment "default/nginx-service cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
ubuntu@vm1:~$ sudo iptables-save | grep  KUBE-SVC-V2OKYYMBY3REGZOG
:KUBE-SVC-V2OKYYMBY3REGZOG - [0:0]
-A KUBE-SERVICES -d 10.43.180.238/32 -p tcp -m comment --comment "default/nginx-service cluster IP" -m tcp --dport 80 -j KUBE-SVC-V2OKYYMBY3REGZOG
-A KUBE-SVC-V2OKYYMBY3REGZOG ! -s 10.42.0.0/16 -d 10.43.180.238/32 -p tcp -m comment --comment "default/nginx-service cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SVC-V2OKYYMBY3REGZOG -m comment --comment "default/nginx-service -> 10.42.0.8:80" -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-LNMZPQ2U2A5TEEGP
-A KUBE-SVC-V2OKYYMBY3REGZOG -m comment --comment "default/nginx-service -> 10.42.1.2:80" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-3Y75O4B4KDVD7TMA
-A KUBE-SVC-V2OKYYMBY3REGZOG -m comment --comment "default/nginx-service -> 10.42.2.2:80" -j KUBE-SEP-Z33JJVRDNG7R4HVW
ubuntu@vm1:~$ sudo iptables-save | grep  KUBE-SEP-LNMZPQ2U2A5TEEGP
:KUBE-SEP-LNMZPQ2U2A5TEEGP - [0:0]
-A KUBE-SEP-LNMZPQ2U2A5TEEGP -s 10.42.0.8/32 -m comment --comment "default/nginx-service" -j KUBE-MARK-MASQ
-A KUBE-SEP-LNMZPQ2U2A5TEEGP -p tcp -m comment --comment "default/nginx-service" -m tcp -j DNAT --to-destination 10.42.0.8:80
-A KUBE-SVC-V2OKYYMBY3REGZOG -m comment --comment "default/nginx-service -> 10.42.0.8:80" -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-LNMZPQ2U2A5TEEGP
ubuntu@vm1:~$ sudo iptables-save | grep  KUBE-SEP-3Y75O4B4KDVD7TMA
:KUBE-SEP-3Y75O4B4KDVD7TMA - [0:0]
-A KUBE-SEP-3Y75O4B4KDVD7TMA -s 10.42.1.2/32 -m comment --comment "default/nginx-service" -j KUBE-MARK-MASQ
-A KUBE-SEP-3Y75O4B4KDVD7TMA -p tcp -m comment --comment "default/nginx-service" -m tcp -j DNAT --to-destination 10.42.1.2:80
-A KUBE-SVC-V2OKYYMBY3REGZOG -m comment --comment "default/nginx-service -> 10.42.1.2:80" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-3Y75O4B4KDVD7TMA
ubuntu@vm1:~$ sudo iptables-save | grep   KUBE-SEP-Z33JJVRDNG7R4HVW
:KUBE-SEP-Z33JJVRDNG7R4HVW - [0:0]
-A KUBE-SEP-Z33JJVRDNG7R4HVW -s 10.42.2.2/32 -m comment --comment "default/nginx-service" -j KUBE-MARK-MASQ
-A KUBE-SEP-Z33JJVRDNG7R4HVW -p tcp -m comment --comment "default/nginx-service" -m tcp -j DNAT --to-destination 10.42.2.2:80
-A KUBE-SVC-V2OKYYMBY3REGZOG -m comment --comment "default/nginx-service -> 10.42.2.2:80" -j KUBE-SEP-Z33JJVRDNG7R4HVW
ubuntu@vm1:~$  

#
# As expected the trafic is load balanced thanks to a set of rules in iptables which defines a separate entry for each target pod.
# Note that there is no SNAT and no need to do so since connection and brought up to services.
#
# KUBE-SERVICES -d 10.43.180.238/32 -m tcp --dport 80---> KUBE-SVC-V2OKYYMBY3REGZOG ----> KUBE-SEP-3Y75O4B4KDVD7TMA (DNAT  to 10.42.1.2:80)
#                                                                                   ----> KUBE-SEP-Z33JJVRDNG7R4HVW (DNAT to 10.42.2.2:80)
#                                                                                   ----> KUBE-SEP-LNMZPQ2U2A5TEEGP (DNAT to 10.42.0.8:80)
# 
```

### Nodeport 

Deploy nodeport service with:
- nodeport port: 30000
- svc port: 80
- container port: 8080

In other KRM words, this is what we have:
```apiVersion: v1
kind: Service
metadata:
  name: nginx-np-service
spec:
  selector:
    app: nginx-np
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30000
  type: NodePort
```
Let's start a new deployment with this nodeport:
```
kubectl apply -f https://raw.githubusercontent.com/robric/multipass-3-node-k8s/main/source/nginx-np-svc.yaml
````
After deployment we have the following:
```console
ubuntu@vm1:~$ kubectl get svc -o wide
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE   SELECTOR
kubernetes         ClusterIP   10.43.0.1       <none>        443/TCP        46h   <none>
nginx-service      ClusterIP   10.43.180.238   <none>        80/TCP         46h   app=nginx
nginx-np-service   NodePort    10.43.143.108   <none>        80:30000/TCP   20s   app=nginx-np
ubuntu@vm1:~$ 
```
Note that nodeport has cluster IP since this is the same logic for intra-cluster communication (i.e. reaching the service from test-pod in previous section).

We see that an additional port is now exposed (30000) for access via nodeport for external connectivity (although this is not recommended).
Now let's have a look at the iptables logic.
```
#
# standard definition of k8s service with Cluster IP  
#
ubuntu@vm1:~$ sudo iptables-save | grep  10.43.143.108 
-A KUBE-SERVICES -d 10.43.143.108/32 -p tcp -m comment --comment "default/nginx-np-service cluster IP" -m tcp --dport 80 -j KUBE-SVC-MUSBZEOMK5UKWKKU
-A KUBE-SVC-MUSBZEOMK5UKWKKU ! -s 10.42.0.0/16 -d 10.43.143.108/32 -p tcp -m comment --comment "default/nginx-np-service cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
ubuntu@vm1:~$ sudo iptables -S KUBE-SVC-MUSBZEOMK5UKWKKU -v
iptables v1.8.7 (nf_tables): chain `KUBE-SVC-MUSBZEOMK5UKWKKU' in table `filter' is incompatible, use 'nft' tool.

ubuntu@vm1:~$ sudo iptables -t nat -S KUBE-SVC-MUSBZEOMK5UKWKKU -v
-N KUBE-SVC-MUSBZEOMK5UKWKKU
-A KUBE-SVC-MUSBZEOMK5UKWKKU ! -s 10.42.0.0/16 -d 10.43.143.108/32 -p tcp -m comment --comment "default/nginx-np-service cluster IP" -m tcp --dport 80 -c 0 0 -j KUBE-MARK-MASQ
-A KUBE-SVC-MUSBZEOMK5UKWKKU -m comment --comment "default/nginx-np-service -> 10.42.0.10:8080" -m statistic --mode random --probability 0.33333333349 -c 0 0 -j KUBE-SEP-MQVY6GCMKDVFWQIB
-A KUBE-SVC-MUSBZEOMK5UKWKKU -m comment --comment "default/nginx-np-service -> 10.42.1.6:8080" -m statistic --mode random --probability 0.50000000000 -c 0 0 -j KUBE-SEP-6BQ3QHB6G4YIKPPI
-A KUBE-SVC-MUSBZEOMK5UKWKKU -m comment --comment "default/nginx-np-service -> 10.42.2.5:8080" -c 0 0 -j KUBE-SEP-746QLTYFWXTG2Q66
ubuntu@vm1:~$ 
#
# Nodeport 
#
ubuntu@vm1:~$ sudo iptables-save | grep  30000 
-A KUBE-ROUTER-INPUT -p tcp -m comment --comment "allow LOCAL TCP traffic to node ports - LR7XO7NXDBGQJD2M" -m addrtype --dst-type LOCAL -m multiport --dports 30000:32767 -j RETURN
-A KUBE-ROUTER-INPUT -p udp -m comment --comment "allow LOCAL UDP traffic to node ports - 76UCBPIZNGJNWNUZ" -m addrtype --dst-type LOCAL -m multiport --dports 30000:32767 -j RETURN
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/nginx-np-service" -m tcp --dport 30000 -j KUBE-EXT-MUSBZEOMK5UKWKKU
ubuntu@vm1:~$ sudo iptables -t nat -L KUBE-EXT-MUSBZEOMK5UKWKKU -v
Chain KUBE-EXT-MUSBZEOMK5UKWKKU (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  any    any     anywhere             anywhere             /* masquerade traffic for default/nginx-np-service external destinations */
    0     0 KUBE-SVC-MUSBZEOMK5UKWKKU  all  --  any    any     anywhere             anywhere            
ubuntu@vm1:~$ 

The nodeport points to the same "KUBE-SVC-MUSBZEOMK5UKWKKU" rule where NAT is enforced.
```
Here is a diagram that summarizes the added logic of nodeports in iptables.
```

KUBE-SERVICES -d 10.43.180.238/32 -m tcp --dport 80---> KUBE-SVC-MUSBZEOMK5UKWKKU +---> KUBE-SEP-MQVY6GCMKDVFWQIB (DNAT  to 10.42.0.10:8080)
                                                        ^                         |---> KUBE-SEP-6BQ3QHB6G4YIKPPI (DNAT to 10.42.1.6:8080)
                                                        |                         |---> KUBE-SEP-746QLTYFWXTG2Q66 (DNAT to 10.42.2.5:8080)
                                            KUBE-EXT-MUSBZEOMK5UKWKKU
                                                        |                              
KUBE-NODEPORTS -m tcp --dport 30000         -------------

```
Of course, all nodes have the same logic. Here is a capture from vm2.
```
ubuntu@vm2:~$ sudo iptables -t nat -L  KUBE-EXT-MUSBZEOMK5UKWKKU -v
Chain KUBE-EXT-MUSBZEOMK5UKWKKU (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  any    any     anywhere             anywhere             /* masquerade traffic for default/nginx-np-service external destinations */
    0     0 KUBE-SVC-MUSBZEOMK5UKWKKU  all  --  any    any     anywhere             anywhere            
ubuntu@vm2:~$ sudo iptables -t nat -L  KUBE-SVC-MUSBZEOMK5UKWKKU
Chain KUBE-SVC-MUSBZEOMK5UKWKKU (2 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  tcp  -- !fiveg-host-24-node4/16  10.43.143.108        /* default/nginx-np-service cluster IP */ tcp dpt:http
KUBE-SEP-MQVY6GCMKDVFWQIB  all  --  anywhere             anywhere             /* default/nginx-np-service -> 10.42.0.10:8080 */ statistic mode random probability 0.33333333349
KUBE-SEP-6BQ3QHB6G4YIKPPI  all  --  anywhere             anywhere             /* default/nginx-np-service -> 10.42.1.6:8080 */ statistic mode random probability 0.50000000000
KUBE-SEP-746QLTYFWXTG2Q66  all  --  anywhere             anywhere             /* default/nginx-np-service -> 10.42.2.5:8080 */
ubuntu@vm2:~$ 
```
## Metalb Integration on 3 node cluster

### Target Design

We're now using the external interface to the current networking thanks to a vlan (vlan 100).

```
            +----------------------------------------------+    
            |                   Cluster                    |    
            | +------------+ +------------+ +------------+ |   
            | |     vm1    | |    vm2     | |    vm3     | |    
            | +----ens3----+ +---ens3-----+ +---ens3-----+ |   
            |      |  |.1/24        |.2/24         |.3/24  |    
            +------|--|-------------|--------------|-------+    
                   |  |             |              |                     
                   |  |             |              |             
                   |  |             |              |              
                   |  |---------vlan 100 (external)-----
                   |              10.123.123.0/24   |              
                   |                                |     
                   |                          [mpqemubr0.100]          
                   |                         10.123.123.254/24          
                   |                                                                       
             ------|------ native vlan (internal)------
                        |         10.65.94.0/24        
                        |
                        |
                 [mpqemubr0]
                 10.65.94.1/24
```
This should be there after deployment, but in case this has been lost (reboot), you may execute the following script:
```
curl -sSL https://raw.githubusercontent.com/robric/multipass-3-node-k8s/main/source/external-net.sh | sh
```

### Metalb Deployment

We'll use info from https://metallb.universe.tf/installation. We also choose the k8s/FRR version because we're a bunch of network nerds.
To deploy metalb, just apply the following manifest in master (vm1):
``` 
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-frr-k8s.yaml
```

### Test in L2 mode

#### Deployment with 3 pods and Single VIP in the external network

We want to expose a dedicated load balancer IP=10.123.123.100 outside the cluster thanks to the external vlan.

```
           +-----------------------------------------------+    
            |                   Cluster                     |    
            | +------------+ +------------+  +------------+ |   
            | |   nginx-1  | |   nginx-2  |  |   nginx-3  | |
            | |            | |            |  |            | |
            | |    vm1     | |    vm2     |  |     vm3    | |
            | |            | |            |  |            | |
            | +----ens3.100+ +---ens3.100-+ +---ens3.100--+ |   
            |        |             |              |         |    
            +--------|-------------|--------------|---------+    
                     |             |              |  
                [ ========= VIP = 10.123.123.100:80 =========]                  
                     |             |              |             
                     |             |              |              
                  ---+--+------vlan 100 (external)+-----
                        |         10.123.123.0/24         
                        |
                        |
                 [mpqemubr0.100]
                 10.123.123.1/24
```
Deploy the metalb service in L2 mode. It will reach the same pods as the basic nginx service thanks to selector.

```
kubectl apply -f https://raw.githubusercontent.com/robric/multipass-3-node-k8s/main/source/nginx-mlb-svc.yaml
```
We're having now a new service of Type LoadBalancer which has an external IP.

```console
ubuntu@vm1:~$ kubectl  get svc -o wide
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE    SELECTOR
kubernetes             ClusterIP      10.43.0.1       <none>           443/TCP        4d1h   <none>
nginx-service          ClusterIP      10.43.180.238   <none>           80/TCP         4d1h   app=nginx
nginx-np-service       NodePort       10.43.143.108   <none>           80:30000/TCP   2d3h   app=nginx-np
nginx-mlb-l2-service   LoadBalancer   10.43.159.55    10.123.123.100   80:30329/TCP   23h    app=nginx-lbl2

ubuntu@vm1:~$ kubectl  get pods -o wide | grep l2
nginx-lbl2-577c9489d-879qj             1/1     Running   0          23h    10.42.1.20   vm2    <none>           <none>
nginx-lbl2-577c9489d-fvk7g             1/1     Running   0          23h    10.42.1.21   vm2    <none>           <none>
nginx-lbl2-577c9489d-fclfn             1/1     Running   0          23h    10.42.0.25   vm1    <none>           <none>
ubuntu@vm1:~$ 

```
We can notice that this an extension of nodeport (a random 30329 port is chosen), the latter being an extension of cluster IP.
We can issue a few request, both from:
- vm1 (the master/worker node)
- An external endpoint such as vm-ext or the host.
```console
ubuntu@vm1:~$ curl 10.123.123.100

 Welcome to NGINX! 
 This is the pod IP address: 10.42.1.21 
 
ubuntu@vm1:~$ curl 10.123.123.100

 Welcome to NGINX! 
 This is the pod IP address: 10.42.0.25 
 
ubuntu@vm-ext:~# curl 10.123.123.100

 Welcome to NGINX! 
 This is the pod IP address: 10.42.1.20 
```
We can check the owner of the VIP thanks to mac inspection, we also see some side effect 
```
#
# From external host/gw:
#

root@fiveg-host-24-node4:~#  arp -na | grep .123.123
? (10.123.123.2) at 52:54:00:c0:87:a0 [ether] on mpqemubr0.100  <============= VIP
? (10.123.123.1) at 52:54:00:c0:87:a0 [ether] on mpqemubr0.100  <============== Whaaat vm1 IP has VM2 mac ???
? (10.123.123.100) at 52:54:00:c0:87:a0 [ether] on mpqemubr0.100 <============= VIP

#
# From vm2:
#

ubuntu@vm2:~$ ip link show dev ens3
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:c0:87:a0 brd ff:ff:ff:ff:ff:ff <========================= This is the mac
#
# If we ping from host to vm1 we see packets transiting via vm2:
#
ubuntu@vm2:~$ sudo tcpdump -evni ens3.100
tcpdump: listening on ens3.100, link-type EN10MB (Ethernet), snapshot length 262144 bytes
05:59:14.860278 52:54:00:e4:50:da > 52:54:00:c0:87:a0, ethertype IPv4 (0x0800), length 98: (tos 0x0, ttl 64, id 32968, offset 0, flags [DF], proto ICMP (1), length 84)
    10.123.123.254 > 10.123.123.1: ICMP echo request, id 240, seq 1, length 64
#
# We can check that proxy arp is disabled - the "speaker" pod from metallb is actually the one that is managing ARP. 
#
ubuntu@vm2:~$ cat /proc/sys/net/ipv4/conf/ens3.100/proxy_arp
0
# 
# The problem got fixed after changing the mask for the metallb VIP pool. This avoids any overlap with existing IP of VMs. 
#
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: external-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.123.123.64/24  ==========>  Use 10.123.123.64/26 instead !!!!
```
Here is the bulk of iptables for bookkeeping. Just look at the diagram right after: the lb is basically another branch to the external service associated with the nodeport.
```
ubuntu@vm1:~$ sudo iptables-save | grep  10.123.123.100
-A KUBE-SERVICES -d 10.123.123.100/32 -p tcp -m comment --comment "default/nginx-mlb-l2-service loadbalancer IP" -m tcp --dport 80 -j KUBE-EXT-W47NQ5DDJKUWFTVY
ubuntu@vm1:~$ sudo iptables-save | grep KUBE-EXT-W47NQ5DDJKUWFTVY
:KUBE-EXT-W47NQ5DDJKUWFTVY - [0:0]
-A KUBE-EXT-W47NQ5DDJKUWFTVY -m comment --comment "masquerade traffic for default/nginx-mlb-l2-service external destinations" -j KUBE-MARK-MASQ
-A KUBE-EXT-W47NQ5DDJKUWFTVY -j KUBE-SVC-W47NQ5DDJKUWFTVY
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/nginx-mlb-l2-service" -m tcp --dport 30329 -j KUBE-EXT-W47NQ5DDJKUWFTVY
-A KUBE-SERVICES -d 10.123.123.100/32 -p tcp -m comment --comment "default/nginx-mlb-l2-service loadbalancer IP" -m tcp --dport 80 -j KUBE-EXT-W47NQ5DDJKUWFTVY
ubuntu@vm1:~$ sudo iptables-save | grep 10.43.159.55
-A KUBE-SERVICES -d 10.43.159.55/32 -p tcp -m comment --comment "default/nginx-mlb-l2-service cluster IP" -m tcp --dport 80 -j KUBE-SVC-W47NQ5DDJKUWFTVY
-A KUBE-SVC-W47NQ5DDJKUWFTVY ! -s 10.42.0.0/16 -d 10.43.159.55/32 -p tcp -m comment --comment "default/nginx-mlb-l2-service cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
ubuntu@vm1:~$ sudo iptables-save | grep KUBE-SVC-W47NQ5DDJKUWFTVY
:KUBE-SVC-W47NQ5DDJKUWFTVY - [0:0]
-A KUBE-EXT-W47NQ5DDJKUWFTVY -j KUBE-SVC-W47NQ5DDJKUWFTVY
-A KUBE-SERVICES -d 10.43.159.55/32 -p tcp -m comment --comment "default/nginx-mlb-l2-service cluster IP" -m tcp --dport 80 -j KUBE-SVC-W47NQ5DDJKUWFTVY
-A KUBE-SVC-W47NQ5DDJKUWFTVY ! -s 10.42.0.0/16 -d 10.43.159.55/32 -p tcp -m comment --comment "default/nginx-mlb-l2-service cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SVC-W47NQ5DDJKUWFTVY -m comment --comment "default/nginx-mlb-l2-service -> 10.42.0.25:8080" -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-H3SFLVALEZ5LECV3
-A KUBE-SVC-W47NQ5DDJKUWFTVY -m comment --comment "default/nginx-mlb-l2-service -> 10.42.1.20:8080" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-U6NBO3SO7ES56QIU
-A KUBE-SVC-W47NQ5DDJKUWFTVY -m comment --comment "default/nginx-mlb-l2-service -> 10.42.1.21:8080" -j KUBE-SEP-UV3QYGUSRGNZ7JSO
ubuntu@vm1:~$ 
```
In more human-friendly ways:
```
KUBE-SERVICES -d 10.43.159.55/32 -p tcp --dport 80---> KUBE-SVC-W47NQ5DDJKUWFTVY +--> DNAT  to 10.42.0.25:8080 / KUBE-SEP-H3SFLVALEZ5LECV3 
                                                         ^                       |--> DNAT  to 10.42.1.20:8080 / KUBE-SEP-U6NBO3SO7ES56QIU
                                                         |                       |--> DNAT  to 10.42.1.21:8080 / KUBE-SEP-UV3QYGUSRGNZ7JSO
                                            KUBE-EXT-W47NQ5DDJKUWFTVY
                                                        ^   ^                              
KUBE-NODEPORTS -m tcp --dport 30329         ------------|   |
                                                            | 
                                                            | 
KUBE-SERVICES -d 10.123.123.100/32 -p tcp --dport 80 -------|
```


#### "in-cluster" trafic to external VIP 10.123.123.100

It is interesting to understand what happens when trafic is sent to the VIP from within the cluster.
Indeed vm1 has ARP entry 10.123.123.100 for vm2... So it is tempting to say that that the request may be sent to VM2... but no it won't !!! iptables will intercept it.

```
#
# From vm1:
#

ubuntu@vm1:~$ arp -na
? (10.123.123.100) at 52:54:00:c0:87:a0 [ether] on ens3.100  <=================== this is vm2 mac address

ubuntu@vm1:~$  curl 10.123.123.100

 Welcome to NGINX! 
 This is the pod IP address: 10.42.1.20 
 
#
#  Trace iptables activity in vm1 by checking the counter which increases from 7 to 8 packets (these are slow path packets -
#  there are more packets than that).
#  Use TCPdump on vm2 to trace packets with the VIP destination address 10.123.123.100 
#

ubuntu@vm1:~$ sudo nft  list table nat | grep 123.100
                meta l4proto tcp ip daddr 10.123.123.100  tcp dport 80 counter packets 7 bytes 420 jump KUBE-EXT-W47NQ5DDJKUWFTVY

ubuntu@vm1:~$ curl 10.123.123.100

 Welcome to NGINX! 
 This is the pod IP address: 10.42.1.21 
 
ubuntu@vm1:~$ sudo nft  list table nat | grep 123.100
                meta l4proto tcp ip daddr 10.123.123.100  tcp dport 80 counter packets 8 bytes 480 jump KUBE-EXT-W47NQ5DDJKUWFTVY


ubuntu@vm2:~$ sudo tcpdump -evni ens3 "tcp and host 10.123.123.100"
tcpdump: listening on ens3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel
ubuntu@vm2:~$ 

#
# We can verify that this is the same for trafic from pods.
#

ubuntu@vm1:~$ kubectl exec -it test-pod -- curl 10.123.123.100

 Welcome to NGINX! 
 This is the pod IP address: 10.42.0.25 
 
```

#### Test of  "externalTrafficPolicy: Local" with 6 pods 

We're slightly changing the previous and add some replicas (6) so we can check the load balancing within a node.

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-mlb-l2-service
[...]
spec:
  externalTrafficPolicy: Local
[...]
```

```console
ubuntu@vm1:~$  kubectl apply -f https://raw.githubusercontent.com/robric/multipass-3-node-k8s/main/source/nginx-mlb-svc-local.yaml
service/nginx-mlb-l2-service configured
ipaddresspool.metallb.io/external-pool unchanged
l2advertisement.metallb.io/l2-metalb unchanged
configmap/nginx-conf unchanged
deployment.apps/nginx-lbl2 configured
ubuntu@vm1:~$
ubuntu@vm1:~$ kubectl get pods -o wide 
NAME                                   READY   STATUS    RESTARTS   AGE    IP           NODE   NOMINATED NODE   READINESS GATES
[...]
nginx-lbl2-577c9489d-879qj             1/1     Running   0          26h    10.42.1.20   vm2    <none>           <none>
nginx-lbl2-577c9489d-fvk7g             1/1     Running   0          26h    10.42.1.21   vm2    <none>           <none>
nginx-lbl2-577c9489d-fclfn             1/1     Running   0          26h    10.42.0.25   vm1    <none>           <none>
nginx-lbl2-577c9489d-kjfdv             1/1     Running   0          30m    10.42.0.26   vm1    <none>           <none>
nginx-lbl2-577c9489d-w7dls             1/1     Running   0          30m    10.42.2.22   vm3    <none>           <none>
nginx-lbl2-577c9489d-bfj8v             1/1     Running   0          30m    10.42.2.21   vm3    <none>           <none>
```
We can verify that the trafic is dispatched solely to pods living in the same master server (vm2): 
 - 10.42.1.20   
 - 10.42.1.21   
```console
root@fiveg-host-24-node4:~# curl 10.123.123.100

 Welcome to NGINX! 
 This is the pod IP address: 10.42.1.21 
 
root@fiveg-host-24-node4:~# curl 10.123.123.100

 Welcome to NGINX! 
 This is the pod IP address: 10.42.1.20 
 
root@fiveg-host-24-node4:~# curl 10.123.123.100

 Welcome to NGINX! 
 This is the pod IP address: 10.42.1.20 
 
root@fiveg-host-24-node4:~# 
etc.
``` 
However, if we try from a pod in vm1... the trafic is dispatched anywhere. 
```console
ubuntu@vm1:~$ kubectl exec -it test-pod -- curl 10.123.123.100

 Welcome to NGINX! 
 This is the pod IP address: 10.42.0.25 
 
ubuntu@vm1:~$ kubectl exec -it test-pod -- curl 10.123.123.100
 
 Welcome to NGINX! 
 This is the pod IP address: 10.42.0.26 
 
...
```

Let's dig in the nat rules to understand what is happening. 

```console
#
# with externalTrafficPolicy: Cluster (default)
#
ubuntu@vm1:~$ sudo iptables -t nat -S KUBE-EXT-W47NQ5DDJKUWFTVY -v 
-N KUBE-EXT-W47NQ5DDJKUWFTVY
-A KUBE-EXT-W47NQ5DDJKUWFTVY -m comment --comment "masquerade traffic for default/nginx-mlb-l2-service external destinations" -c 0 0 -j KUBE-MARK-MASQ
-A KUBE-EXT-W47NQ5DDJKUWFTVY -c 0 0 -j KUBE-SVC-W47NQ5DDJKUWFTVY
#
# with externalTrafficPolicy: Local
#
ubuntu@vm1:~$ sudo iptables -t nat -S KUBE-EXT-W47NQ5DDJKUWFTVY -v 
-N KUBE-EXT-W47NQ5DDJKUWFTVY
-A KUBE-EXT-W47NQ5DDJKUWFTVY -s 10.42.0.0/16 -m comment --comment "pod traffic for default/nginx-mlb-l2-service external destinations" -c 0 0 -j KUBE-SVC-W47NQ5DDJKUWFTVY
-A KUBE-EXT-W47NQ5DDJKUWFTVY -m comment --comment "masquerade LOCAL traffic for default/nginx-mlb-l2-service external destinations" -m addrtype --src-type LOCAL -c 0 0 -j KUBE-MARK-MASQ
-A KUBE-EXT-W47NQ5DDJKUWFTVY -m comment --comment "route LOCAL traffic for default/nginx-mlb-l2-service external destinations" -m addrtype --src-type LOCAL -c 0 0 -j KUBE-SVC-W47NQ5DDJKUWFTVY
-A KUBE-EXT-W47NQ5DDJKUWFTVY -c 0 0 -j KUBE-SVL-W47NQ5DDJKUWFTVY

Two noticable changes in KUBE-EXT-W47NQ5DDJKUWFTVY
 - A test for source in 10.42.0.0.0/16 (= pod IPs) whic calls the KUBE-SVC-W47NQ5DDJKUWFTVY rule which has the 6 pods of the deployment. 
 - The default rule is changed to KUBE-SVL-W47NQ5DDJKUWFTVY which has the 2 local pods.

ubuntu@vm1:~$ sudo iptables -t nat -S KUBE-SVL-W47NQ5DDJKUWFTVY -v 
-N KUBE-SVL-W47NQ5DDJKUWFTVY
-A KUBE-SVL-W47NQ5DDJKUWFTVY -m comment --comment "default/nginx-mlb-l2-service -> 10.42.0.25:8080" -m statistic --mode random --probability 0.50000000000 -c 0 0 -j KUBE-SEP-H3SFLVALEZ5LECV3
-A KUBE-SVL-W47NQ5DDJKUWFTVY -m comment --comment "default/nginx-mlb-l2-service -> 10.42.0.26:8080" -c 0 0 -j KUBE-SEP-JIHH5JQABFO67SHN
ubuntu@vm1:~$ sudo iptables -t nat -S KUBE-SVC-W47NQ5DDJKUWFTVY -v 
-N KUBE-SVC-W47NQ5DDJKUWFTVY
-A KUBE-SVC-W47NQ5DDJKUWFTVY ! -s 10.42.0.0/16 -d 10.43.159.55/32 -p tcp -m comment --comment "default/nginx-mlb-l2-service cluster IP" -m tcp --dport 80 -c 0 0 -j KUBE-MARK-MASQ
-A KUBE-SVC-W47NQ5DDJKUWFTVY -m comment --comment "default/nginx-mlb-l2-service -> 10.42.0.25:8080" -m statistic --mode random --probability 0.16666666651 -c 0 0 -j KUBE-SEP-H3SFLVALEZ5LECV3
-A KUBE-SVC-W47NQ5DDJKUWFTVY -m comment --comment "default/nginx-mlb-l2-service -> 10.42.0.26:8080" -m statistic --mode random --probability 0.20000000019 -c 0 0 -j KUBE-SEP-JIHH5JQABFO67SHN
-A KUBE-SVC-W47NQ5DDJKUWFTVY -m comment --comment "default/nginx-mlb-l2-service -> 10.42.1.20:8080" -m statistic --mode random --probability 0.25000000000 -c 0 0 -j KUBE-SEP-U6NBO3SO7ES56QIU
-A KUBE-SVC-W47NQ5DDJKUWFTVY -m comment --comment "default/nginx-mlb-l2-service -> 10.42.1.21:8080" -m statistic --mode random --probability 0.33333333349 -c 0 0 -j KUBE-SEP-UV3QYGUSRGNZ7JSO
-A KUBE-SVC-W47NQ5DDJKUWFTVY -m comment --comment "default/nginx-mlb-l2-service -> 10.42.2.21:8080" -m statistic --mode random --probability 0.50000000000 -c 0 0 -j KUBE-SEP-5XVJDSQ32SUUTXIC
-A KUBE-SVC-W47NQ5DDJKUWFTVY -m comment --comment "default/nginx-mlb-l2-service -> 10.42.2.22:8080" -c 0 0 -j KUBE-SEP-DTKU7V2IJOC7TTKI
ubuntu@vm1:~$ 
```
Let's try to toggle the  nginx-mlb-l2-service with internalTrafficPolicy to Local.
```
kind: Service
metadata:
[...]
  name: nginx-mlb-l2-service
spec:
[...]
  externalTrafficPolicy: Local
  internalTrafficPolicy: Local              <======================== let's try that 
```
It does not work unfortunately.
```console
#
# The rule is unchanged: the pod subnet is still routed to the KUBE-SVC service (with 6 pods)
#
ubuntu@vm1:~$ sudo iptables -t nat -S KUBE-EXT-W47NQ5DDJKUWFTVY -v 
-N KUBE-EXT-W47NQ5DDJKUWFTVY
-A KUBE-EXT-W47NQ5DDJKUWFTVY -s 10.42.0.0/16 -m comment --comment "pod traffic for default/nginx-mlb-l2-service external destinations" -c 0 0 -j KUBE-SVC-W47NQ5DDJKUWFTVY
-A KUBE-EXT-W47NQ5DDJKUWFTVY -m comment --comment "masquerade LOCAL traffic for default/nginx-mlb-l2-service external destinations" -m addrtype --src-type LOCAL -c 0 0 -j KUBE-MARK-MASQ
-A KUBE-EXT-W47NQ5DDJKUWFTVY -m comment --comment "route LOCAL traffic for default/nginx-mlb-l2-service external destinations" -m addrtype --src-type LOCAL -c 0 0 -j KUBE-SVC-W47NQ5DDJKUWFTVY
-A KUBE-EXT-W47NQ5DDJKUWFTVY -c 0 0 -j KUBE-SVL-W47NQ5DDJKUWFTVY
ubuntu@vm1:~$
#
# There is no magic: curl requests are spread everywhere
#

ubuntu@vm1:~$ kubectl exec -it test-pod -- curl 10.123.123.100

 Welcome to NGINX! 
 This is the pod IP address: 10.42.0.25  <========================= vm1
 
ubuntu@vm1:~$ kubectl exec -it test-pod -- curl 10.123.123.100

 Welcome to NGINX! 
 This is the pod IP address: 10.42.2.21 <========================= vm2
 
ubuntu@vm1:~$
```
#### metallb deployment in hostnetwork

The following manifest deploys 3 pods in hostnetwork, together with VIP 10.123.123.102 in the external network via metallb.

```
kubectl apply -f https://raw.githubusercontent.com/robric/multipass-3-node-k8s/main/source/nginx-mlb-svc-hostnetwork.yaml
```
This works as expected.
```
ubuntu@vm1:~$ kubectl get svc | grep 10.123.123.102
nginx-mlb-l2-hostnet   LoadBalancer   10.43.207.14    10.123.123.102   80:30733/TCP   6m56s
ubuntu@vm1:~$ 
ubuntu@vm1:~$ kubectl  get pods -o wide | grep net
nginx-lbl2-hostnet-6d7754d545-5g8r8    1/1     Running   0          3m22s   10.65.94.95    vm3    <none>           <none>
nginx-lbl2-hostnet-6d7754d545-59d6g    1/1     Running   0          3m22s   10.65.94.199   vm2    <none>           <none>
nginx-lbl2-hostnet-6d7754d545-69kxn    1/1     Running   0          3m22s   10.65.94.238   vm1    <none>           <none>
ubuntu@vm1:~$ 
root@fiveg-host-24-node4:~# curl 10.123.123.102

 Welcome to NGINX! 
 This is the pod IP address: 10.65.94.95 

root@fiveg-host-24-node4:~# curl 10.123.123.102

 Welcome to NGINX! 
 This is the pod IP address: 10.65.94.199 
 
root@fiveg-host-24-node4:~# 
```
There is not much change in NAT logic compared with 
```
ubuntu@vm1:~$ sudo iptables-save |grep 10.123.123.102
-A KUBE-SERVICES -d 10.123.123.102/32 -p tcp -m comment --comment "default/nginx-mlb-l2-hostnet loadbalancer IP" -m tcp --dport 80 -j KUBE-EXT-CDEB5OJ5KJNVMLM6
ubuntu@vm1:~$ sudo iptables -t nat -L KUBE-EXT-CDEB5OJ5KJNVMLM6 -n 
Chain KUBE-EXT-CDEB5OJ5KJNVMLM6 (2 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  all  --  0.0.0.0/0            0.0.0.0/0            /* masquerade traffic for default/nginx-mlb-l2-hostnet external destinations */
KUBE-SVC-CDEB5OJ5KJNVMLM6  all  --  0.0.0.0/0            0.0.0.0/0           
ubuntu@vm1:~$ sudo iptables -t nat -L KUBE-SVC-CDEB5OJ5KJNVMLM6 -n 
Chain KUBE-SVC-CDEB5OJ5KJNVMLM6 (2 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  tcp  -- !10.42.0.0/16         10.43.207.14         /* default/nginx-mlb-l2-hostnet cluster IP */ tcp dpt:80
KUBE-SEP-BRCASNC5D2TNKFYB  all  --  0.0.0.0/0            0.0.0.0/0            /* default/nginx-mlb-l2-hostnet -> 10.65.94.199:8080 */ statistic mode random probability 0.33333333349
KUBE-SEP-7VFNSJIQ2K6SELCJ  all  --  0.0.0.0/0            0.0.0.0/0            /* default/nginx-mlb-l2-hostnet -> 10.65.94.238:8080 */ statistic mode random probability 0.50000000000
KUBE-SEP-DYIL5FTYGTTVEK47  all  --  0.0.0.0/0            0.0.0.0/0            /* default/nginx-mlb-l2-hostnet -> 10.65.94.95:8080 */
ubuntu@vm1:~$ sudo iptables -t nat -L KUBE-SEP-BRCASNC5D2TNKFYB  -n 
Chain KUBE-SEP-BRCASNC5D2TNKFYB (1 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  all  --  10.65.94.199         0.0.0.0/0            /* default/nginx-mlb-l2-hostnet */
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* default/nginx-mlb-l2-hostnet */ tcp to:10.65.94.199:8080
ubuntu@vm1:~$ 
```

The change is related to KUBE-SEP instantiation with DNAT happens to node:8080 address -which is actually the pod address in the case of hostnetwork-. 
This is business as usual.


#### Source NAT (masquerade) enforcement for incoming trafic in default mode

There is a subtle behavior change when playing with externalTrafficPolicy related to the enforcment of SNAT:
- externalTrafficPolicy: Cluster (default)
Source NAT is enforced for incoming trafic from external sources. This configuration permits return trafic to be tromboned via the server -identified by SRC NAT IP address- that handles the VIP.   
- externalTrafficPolicy: Local 
Source NAT is NOT enforced for incoming trafic from external sources. In this case, since the server is the same there is no need for NAT.

(NOTE): This change of behavior is actually important since more complex integration work/won't work (see section [Metallb with IPSEC and SCTP](#Metallb-with-ipsec-strongswan-and-sctp))

```console
ubuntu@vm1:~$ kubectl get pods -o wide | grep vm2
[...]
nginx-lbl2-577c9489d-fvk7g             1/1     Running   0          5d20h   10.42.1.21   vm2    <none>           <none>
ubuntu@vm1:~$ kubectl debug -it nginx-lbl2-577c9489d-fvk7g --image nicolaka/netshoot
[...]

#
#   externalTrafficPolicy: Cluster -------> Source address of curl is Natted to  10.42.1.1
#

 nginx-lbl2-577c9489d-fvk7g  ~  tcpdump -ni eth0
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
09:49:45.535241 IP 10.42.1.1.27659 > 10.42.1.21.8080: Flags [S], seq 568611649, win 64240, options [mss 1460,sackOK,TS val 1343731101 ecr 0,nop,wscale 7], length 0
09:49:45.535298 IP 10.42.1.21.8080 > 10.42.1.1.27659: Flags [S.], seq 3912071880, ack 568611650, win 64308, options [mss 1410,sackOK,TS val 3484627024 ecr 1343731101,nop,wscale 7], length 0

#
#   externalTrafficPolicy: Local -------> Source address is Not Natted 10.123.123.100
#

09:50:54.098279 IP 10.123.123.254.38124 > 10.42.1.21.8080: Flags [S], seq 4011661290, win 64240, options [mss 1460,sackOK,TS val 1343799665 ecr 0,nop,wscale 7], length 0
09:50:54.098324 IP 10.42.1.21.8080 > 10.123.123.254.38124: Flags [S.], seq 1190145475, ack 4011661291, win 64308, options [mss 1410,sackOK,TS val 3766712946 ecr 1343799665,nop,wscale 7], length 0

``` 

This is due to masquerading (-j MASQ) configured in iptables for the EXT rule

```
#
#   externalTrafficPolicy: Cluster -------> rule KUBE-MARK-MASQ - which ultimately calls -j MASQ action - is matched (pkts 9)
#

ubuntu@vm2:~$ sudo iptables -t nat -L KUBE-EXT-W47NQ5DDJKUWFTVY -n -v
Chain KUBE-EXT-W47NQ5DDJKUWFTVY (2 references)
 pkts bytes target     prot opt in     out     source               destination         
--> 9   540 KUBE-MARK-MASQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* masquerade traffic for default/nginx-mlb-l2-service external destinations */
--> 9   540 KUBE-SVC-W47NQ5DDJKUWFTVY  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
ubuntu@vm2:~$ 

#
#   externalTrafficPolicy: Local -------> KUBE-SVL-W47NQ5DDJKUWFTVY is called directly without checking KUBE-MARK-MASK 
#                                         Indeed, there is an extra condition "ADDRTYPE match src-type LOCAL" which 
#                                         checks whether source is local
#

ubuntu@vm2:~$ sudo iptables -t nat -L KUBE-EXT-W47NQ5DDJKUWFTVY -n -v
Chain KUBE-EXT-W47NQ5DDJKUWFTVY (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SVC-W47NQ5DDJKUWFTVY  all  --  *      *       10.42.0.0/16         0.0.0.0/0            /* pod traffic for default/nginx-mlb-l2-service external destinations */
    0     0 KUBE-MARK-MASQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* masquerade LOCAL traffic for default/nginx-mlb-l2-service external destinations */ ADDRTYPE match src-type LOCAL
    0     0 KUBE-SVC-W47NQ5DDJKUWFTVY  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* route LOCAL traffic for default/nginx-mlb-l2-service external destinations */ ADDRTYPE match src-type LOCAL
--> 2   120 KUBE-SVL-W47NQ5DDJKUWFTVY  all  --  *      *       0.0.0.0/0            0.0.0.0/0

ubuntu@vm2:~$ 

For fun, we can that if we use an IP that is local to the node (but external to the pod network), then the processing is different (use of masquerade and default cluster-wide KUBE-SVC instead of local KUBE-SVL)

ubuntu@vm2:~$ curl 10.123.123.100 --interface 10.65.94.199

 Welcome to NGINX! 
 This is the pod IP address: 10.42.2.22 
 
ubuntu@vm2:~$ sudo iptables -t nat -L KUBE-EXT-W47NQ5DDJKUWFTVY -n -v
Chain KUBE-EXT-W47NQ5DDJKUWFTVY (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SVC-W47NQ5DDJKUWFTVY  all  --  *      *       10.42.0.0/16         0.0.0.0/0            /* pod traffic for default/nginx-mlb-l2-service external destinations */
--> 1   120 KUBE-MARK-MASQ  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* masquerade LOCAL traffic for default/nginx-mlb-l2-service external destinations */ ADDRTYPE match src-type LOCAL
--> 1   120 KUBE-SVC-W47NQ5DDJKUWFTVY  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* route LOCAL traffic for default/nginx-mlb-l2-service external destinations */ ADDRTYPE match src-type LOCAL
    2   120 KUBE-SVL-W47NQ5DDJKUWFTVY  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
ubuntu@vm2:~$  
``` 

#### External Routing from within pods

This is where things gets a bit more complicated: assume pod can be used to forward traffic (e.g. the pod terminates a Tunnel like IPSEC). We have sources, which raises concerns on how the trafic is routed back. 

We're using "externalTrafficPolicy: Cluster" which is the default mode. 
```
+-------------------------+ +----------+ +----------+     
|                         | |          | |          |     
|  +-------------------+  | |          | |          |     
|  |  test-pod-vm1     |  | |          | |          |     
|  | +--------------+  |  | |          | |          |     
|  | | br-inpod     |  |  | |          | |          |     
|  | |11.11.11.11/24|  |  | |          | |          |     
|  | +--------------+  |  | |          | |          |     
|  |      eth0         |  | |          | |          |     
|  +------+---+--------+  | |          | |          |     
|         |   |           | |          | |          |     
|         |   |           | |          | |          |     
|         |   |           | |          | |          |     
|         +---+           | |          | |          |     
|                         | |          | |          |     
|    +----------+         | |+--------+| |+--------+|     
|    |  nginx1  |         | || nginx2 || || nginx3 ||     
|    |          |         | ||        || ||        ||     
|    +----------+         | |+--------+| |+--------+|     
|                         | |          | |          |     
|                         | |          | |          |     
+-------------------------+ +----------+ +----------+     
           VM1                  VM2           VM3      
```
Spawn netshoot pod on vm1. 
```
kubectl apply -f https://raw.githubusercontent.com/robric/multipass-3-node-k8s/main/source/test-pod-vm1.yaml
```
And let's add an interface within this pod so we can generate "external request" from this pod to the metalb VIP.
```
ubuntu@vm1:~$ kubectl describe pod test-pod-vm1 | grep container
    Container ID:  containerd://87f8f7961ee33c3c2d439621aa88bf4a882ac2538bec7526039a5e21e494ee14

ubuntu@vm1:~$ sudo ctr c info  87f8f7961ee33c3c2d439621aa88bf4a882ac2538bec7526039a5e21e494ee14  |  grep -C 5 pid
                }
            },
            "cgroupsPath": "kubepods-besteffort-podbf227f8b_8058_49b8_be42_1b73feb8a803.slice:cri-containerd:87f8f7961ee33c3c2d439621aa88bf4a882ac2538bec7526039a5e21e494ee14",
            "namespaces": [
                {
                    "type": "pid"
                },
                {
                    "type": "ipc",
                    "path": "/proc/555560/ns/ipc"
                },
ubuntu@vm1:~$
ubuntu@vm1:~$ sudo nsenter -t 555560 -n
root@vm1:/home/ubuntu# ip link add br-inpod type bridge
root@vm1:/home/ubuntu# ip addr add 11.11.11.11/24 dev br-inpod 
root@vm1:/home/ubuntu# ip link set up dev br-inpod
root@vm1:/home/ubuntu# ip route show
default via 10.42.0.1 dev eth0 
10.42.0.0/24 dev eth0 proto kernel scope link src 10.42.0.27 
10.42.0.0/16 via 10.42.0.1 dev eth0 
11.11.11.0/24 dev br-inpod proto kernel scope link src 11.11.11.11 
root@vm1:/home/ubuntu# 

root@vm1:/home/ubuntu# curl 10.123.123.100

 Welcome to NGINX! 
 This is the pod IP address: 10.42.0.25 
 
root@vm1:/home/ubuntu# curl 10.123.123.100 --interface 11.11.11.11 
^C
root@vm1:/home/ubuntu# 

#
# This breaks -as expected- since there is no route back to 11.11.11.0/24 in the kernel
#

```

We can fix that provided that you have netadmin rights in default network ns - which is very bad  practice - (but that's just a test).

```

#
# add route to the pod for the br-inpod subnet
#
ubuntu@vm1:~$ kubectl get pods test-pod-vm1 -o wide
NAME           READY   STATUS    RESTARTS   AGE   IP           NODE   NOMINATED NODE   READINESS GATES
test-pod-vm1   1/1     Running   0          21h   10.42.0.27   vm1    <none>           <none>

ubuntu@vm1:~$ sudo ip route add 11.11.11.0/24 via 10.42.0.27 

#
# curl from pod works now
#

root@vm1:/home/ubuntu# curl 10.123.123.100 --interface 11.11.11.11 

 Welcome to NGINX! 
 This is the pod IP address: 10.42.0.25 
 
root@vm1:/home/ubuntu# 
```

#### Non fate-sharing deployment with metallb and  multiple VIPs

Here we're deploying a slightly more complex setup to expose 2 Metalb VIPs, while making sure there is no zone sharing.
- Two zone labels for computes: vm1 and vm2 in zone1 and vm3 in zone2.
- Two independant deployments nginx-zone1/2 mapped to their respective zone (nodeselector affinity)
- Two distincts VIPs exposed via metalb: vip1=10.123.123.201 and vip=210.123.123.202, each mapped to distinct zone

The following drawing represents the test setup:

```
                                                 |                                                  
                                        zone 1   |   zone2                                          
                                                 |                                                  
                                                 |                                                  
           +-----------------+ +----------------+|+----------------+                                
           |                 | |                |||                |                                
           |                 | |                |||                |                                
           |   +----------+  | |   +---------+  |||   +---------+  |                                
           |   |  nginx1  |  | |   | nginx2  |  |||   | nginx3  |  |                                
           |   |  zone 1  |  | |   | zone1   |  |||   | zone2   |  |                                
           |   |          |  | |   |         |  |||   |         |  |                                
           |   +----------+  | |   +---------+  |||   +---------+  |                                
           |                 | |                |||                |                                
           |                 | |                |||                |                                
           |                 | |                |||                |                                
           |                 | |                |||                |                                
           |      zone1      | |    zone1       |||   zone2        |                                
           |                 | |                |||                |                                
           |                 | |                |||                |                                
           |                 | |                |||                |                                
           |VM1              | |VM2             ||| VM3            |                                
           +-----------------+ +----------------+|+----------------+                                
                                                 |                                                  
                                                 |                                                  
            <------        VIP1     --------->   | <-   VIP2   ->                                   
                      10.123.123.201             |  10.123.123.202    

```
To achieve this, nodes are labelled based on two distincts zones:  
```
kubectl label nodes vm1 zone=zone1
kubectl label nodes vm2 zone=zone1
kubectl label nodes vm3 zone=zone2
```

Apply the manifest for metallb service and app deployments

```
kubectl apply -f https://raw.githubusercontent.com/robric/multipass-3-node-k8s/main/source/nginx-mlb-2-vips.yaml
```

For the sake of simplicity here what is defined in service/metalb - only zone1 displayed for simplicity -.

```
#
# Define a first service nginx-zone1 with selector nginx-zone1 (app deployment steered to zone1)
#
apiVersion: v1
kind: Service
metadata:
  name: nginx-mlb-l2-zone1
  annotations:
    metallb.universe.tf/address-pool: external-pool-zone1
spec:
  selector:
    app: nginx-zone1
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
---


#
# Define a pool for zone1 (ipaddresspool) with vip 10.123.123.201
#

apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: external-pool-zone1
  namespace: metallb-system
spec:
  addresses:
  - 10.123.123.201/32
---

#
# Use an L2advertisement CRD to control the management of the VIP. 
# This maps with the ip pool for zone1 (external-pool-zone1) together with a nodeselector to zone1.
#

apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-metalb-zone1
  namespace: metallb-system
spec:
  ipAddressPools:
  - external-pool-zone1
  interfaces:
  - ens3.100
  nodeSelectors:
  - matchLabels:
      zone: "zone1"
```

After configuration, we can see 3 pods, services and deployments mapped to appropriate zones.

```console
ubuntu@vm1:~$ kubectl get pods -o wide | grep zone
nginx-zone2-7f8b84d998-s4g75           1/1     Running   0          3h2m    10.42.2.36   vm3    <none>           <none>
nginx-zone1-58ccdbd4b8-p49gv           1/1     Running   0          3h2m    10.42.0.37   vm1    <none>           <none>
nginx-zone1-58ccdbd4b8-5m5fh           1/1     Running   0          3h2m    10.42.1.35   vm2    <none>           <none>
ubuntu@vm1:~$ kubectl get svc -o wide | grep zone
nginx-mlb-l2-zone2     LoadBalancer   10.43.3.184     10.123.123.202   80:31405/TCP   3h3m    app=nginx-zone2
nginx-mlb-l2-zone1     LoadBalancer   10.43.165.8     10.123.123.201   80:30980/TCP   3h3m    app=nginx-zone1
ubuntu@vm1:~$ kubectl get deployments.apps -o wide | grep zone
nginx-zone2           1/1     1            1           3h3m    nginx        nginx:latest   app=nginx-zone2
nginx-zone1           2/2     2            2           3h3m    nginx        nginx:latest   app=nginx-zone1
ubuntu@vm1:~$
```

The requests to each VIPs are properly directed to the correct zone.

```console
root@fiveg-host-24-node4:~# curl 10.123.123.201 ------------> request to VIP zone 1

 Welcome to NGINX! 
 Here are the IP address:port tuples for:
  - nginx server => 10.42.1.35:8080             ------------> pod in vm2 / zone1
  - http client  => 10.42.0.0:51172 

root@fiveg-host-24-node4:~# curl 10.123.123.201 ------------> request to VIP zone 1

 Welcome to NGINX! 
 Here are the IP address:port tuples for:
  - nginx server => 10.42.0.37:8080            ------------> pod in vm1 / zone1 
  - http client  => 10.42.0.1:18972 

root@fiveg-host-24-node4:~# curl 10.123.123.202 ------------> request to VIP zone 2

 Welcome to NGINX! 
 Here are the IP address:port tuples for:
  - nginx server => 10.42.2.36:8080             ------------> pod in vm3 / zone2
  - http client  => 10.42.0.0:3152 

```
If we investigate each VIP, we can see that both VIPs are managed by the same host (hence same zone).
This does not seem correct considering the nodeselector mapping attached to L2advertisement.
```console
root@fiveg-host-24-node4:~# arp -na | grep 10.123
? (10.123.123.2) at 52:54:00:c0:87:a0 [ether] on mpqemubr0.100
? (10.123.123.1) at 52:54:00:e2:c3:ec [ether] on mpqemubr0.100
? (10.123.123.202) at 52:54:00:e2:c3:ec [ether] on mpqemubr0.100
? (10.123.123.201) at 52:54:00:e2:c3:ec [ether] on mpqemubr0.100
? (10.123.123.3) at 52:54:00:db:2b:ce [ether] on mpqemubr0.100
root@fiveg-host-24-node4:~# 

#
#     We see that 10.123.123.201 and 10.123.123.202 are bound to the same MAC 52:54:00:e2:c3:ec,
#     which is VM1. This does not look right !
#

```
Let's see what happens if we update the services with "externalTraficPolicy = local" since landing on vm1 for zone2 would not allow forwarding to vm3 in zone2.
```
kubectl apply -f https://raw.githubusercontent.com/robric/multipass-3-node-k8s/main/source/nginx-mlb-2-vips-local.yaml
```

It actually works as expected. We can achieve non fate-sharing deployments:
- vip 10.123.123.201 is handled by 52:54:00:e2:c3:ec = vm1 in zone1
- vip 10.123.123.202 is handled by 52:54:00:db:2b:ce = vm3 in zone2

```console
root@fiveg-host-24-node4:~# curl 10.123.123.202 ------------> request to VIP zone 2

 Welcome to NGINX! 
 Here are the IP address:port tuples for:
  - nginx server => 10.42.2.36:8080            ------------> pod in vm3 / zone2
  - http client  => 10.123.123.254:44492 

root@fiveg-host-24-node4:~# curl 10.123.123.201 ------------> request to VIP zone 1

 Welcome to NGINX! 
 Here are the IP address:port tuples for:
  - nginx server => 10.42.0.37:8080             ------------> pod in vm1 / zone1
  - http client  => 10.123.123.254:34188 

root@fiveg-host-24-node4:~# arp -na | grep 10.123

? (10.123.123.202) at 52:54:00:db:2b:ce [ether] on mpqemubr0.100
? (10.123.123.201) at 52:54:00:e2:c3:ec [ether] on mpqemubr0.100
root@fiveg-host-24-node4:~# 

```

#### Metallb compliance with SCTP

Metallb works in conjunction with sctp (after this is a control plane)

First install SCTP tools on each VM and the host (this will take care of drivers and make it easy to test).
``` 
multipass exec vm1 -- sudo apt install lksctp-tools -y
multipass exec vm2 -- sudo apt install lksctp-tools -y
multipass exec vm3 -- sudo apt install lksctp-tools -y
multipass exec vm-ext -- sudo apt install lksctp-tools -y
sudo apt install lksctp-tools -y
```
Then launch the deployment of the sctp service and load balancer (VIP 10.123.123.101 port 10000).
```
kubectl apply -f https://raw.githubusercontent.com/robric/multipass-3-node-k8s/main/source/sctp-mlb-svc.yaml
```
Now we can test from the host (i.e. external).

```
#
# That worked !!!
#
root@fiveg-host-24-node4:~# sctp_test -H 10.123.123.254 -h  10.123.123.101 -p 10000 -s
remote:addr=10.123.123.101, port=webmin, family=2
local:addr=10.123.123.254, port=0, family=2
seed = 1718815477

Starting tests...
        socket(SOCK_SEQPACKET, IPPROTO_SCTP)  ->  sk=3
        bind(sk=3, [a:10.123.123.254,p:0])  --  attempt 1/10
Client: Sending packets.(1/10)
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=1844700133
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=1188217067
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=1427423053
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=1690014943
[...]

#
# 
#
```

We can use conntrack command to inspect sctp translations / events, while running multiple sctp connection test (previous command).

```
#
# First you need to understand which VM is carrying the VIP. This is where nat is enforced.
#

root@fiveg-host-24-node4:~# arp -na | grep 123.101
? (10.123.123.101) at 52:54:00:db:2b:ce [ether] on mpqemubr0.100
root@fiveg-host-24-node4:~# arp -na | grep  52:54:00:db:2b:ce
? (10.65.94.95) at 52:54:00:db:2b:ce [ether] on mpqemubr0
? (10.123.123.101) at 52:54:00:db:2b:ce [ether] on mpqemubr0.100
? (10.123.123.202) at 52:54:00:db:2b:ce [ether] on mpqemubr0.100
? (10.123.123.3) at 52:54:00:db:2b:ce [ether] on mpqemubr0.100 <--------- This VM3 !
root@fiveg-host-24-node4:~# 

#
# Let's inspect what is happening at vm3  
#

ubuntu@vm3:~$ sudo apt install conntrack
Reading package lists... Done
Building dependency tree... Done
[...]
ubuntu@vm3:~$ sudo conntrack -L -p sctp
sctp     132 6 CLOSED src=10.123.123.254 dst=10.123.123.101 sport=38211 dport=10000 src=10.42.2.37 dst=10.42.2.1 sport=9999 dport=1387 [ASSURED] mark=0 use=1
conntrack v1.4.6 (conntrack-tools): 1 flow entries have been shown.
ubuntu@vm3:~$ 
ubuntu@vm3:~$ sudo conntrack -E -p sctp
    [NEW] sctp     132 10 CLOSED src=10.123.123.254 dst=10.123.123.101 sport=49035 dport=10000 [UNREPLIED] src=10.42.1.36 dst=10.42.2.0 sport=9999 dport=36208
 [UPDATE] sctp     132 3 COOKIE_WAIT src=10.123.123.254 dst=10.123.123.101 sport=49035 dport=10000 src=10.42.1.36 dst=10.42.2.0 sport=9999 dport=36208
 [UPDATE] sctp     132 3 COOKIE_ECHOED src=10.123.123.254 dst=10.123.123.101 sport=49035 dport=10000 src=10.42.1.36 dst=10.42.2.0 sport=9999 dport=36208
 [UPDATE] sctp     132 210 ESTABLISHED src=10.123.123.254 dst=10.123.123.101 sport=49035 dport=10000 src=10.42.1.36 dst=10.42.2.0 sport=9999 dport=36208 [ASSURED]
 [UPDATE] sctp     132 3 SHUTDOWN_SENT src=10.123.123.254 dst=10.123.123.101 sport=49035 dport=10000 src=10.42.1.36 dst=10.42.2.0 sport=9999 dport=36208 [ASSURED]
 [UPDATE] sctp     132 3 SHUTDOWN_ACK_SENT src=10.123.123.254 dst=10.123.123.101 sport=49035 dport=10000 src=10.42.1.36 dst=10.42.2.0 sport=9999 dport=36208 [ASSURED]
 [UPDATE] sctp     132 10 CLOSED src=10.123.123.254 dst=10.123.123.101 sport=49035 dport=10000 src=10.42.1.36 dst=10.42.2.0 sport=9999 dport=36208 [ASSURED]
....

#
# We can see that sctp relies on random ports.
#
```

Let's inspect if sessions are load balanced.
For this purpose, we run a loop of 1000 sctp tests. For some reason, we get only 977 entries in conntrack... no big deal, that's good enough to do some stats.

From the external server (fiveg-host-24-node4): 
``` 
for ((i = 1; i <= 1000; i++)); do     sctp_test -H 10.123.123.254 -h 10.123.123.101 -p 10000 -s -x 1; done
```
Trace new entries (NEW status) on vm3 (=VIP owner) and count for each pod. The file is actually in the log folder of this repo. 
```
ubuntu@vm3:~$ sudo conntrack -E -p sctp -e new > test-1000-sessions.log
ubuntu@vm3:~$

#
# A rapid inspect of the file shows 3 pods IPs: 10.42.1.36, 10.42.2.37 and 10.42.0.38
#

#
# We can simply count each entry to check how load balancing worked.
#
ubuntu@vm3:~$ cat test-1000-sessions.log | wc
    977   13678  155189
ubuntu@vm3:~$ cat test-1000-sessions.log | grep 10.42.2.37 |  wc
    316    4424   50194
ubuntu@vm3:~$ cat test-1000-sessions.log | grep 10.42.0.38 |  wc
    338    4732   53696
ubuntu@vm3:~$ cat test-1000-sessions.log | grep 10.42.1.36 |  wc
    323    4522   51299
ubuntu@vm3:~$

```

CONCLUSION: *We're getting decent statistical distribution of the session on all pods.* . It seems we're not using ipvs (looks like a complex change in k3s unfortunately :-(), since ipvs can do round robin. 


#### Metallb with IPSEC (strongswan) and SCTP

##### Test Setup Installation

In this section, we're testing a more complex integration with dual Metallb services:
- A VIP exposes an external IP for IPSEC termination
- A VIP exposes an external IP for SCTP termination. The SCTP traffic is encrypted via IPSEC in tunnel mode.

The IPSEC is managed by strongswan running in hostnetwork, where tunnel are terminated. This approach simplifies the routing/ipsec policies (ip xfrm policy) back to the remote subnets.

The pod logic is as follow:
- 3 pods for IPSEC control plane (actually daemonset running on each server)
- 6 pods for the SCTP servers so we can test local LB (2 per servers for testing ExternalTrafficPolicy: Local) 

The external VM (vm-ext) is a remote client for SCTP over IPSEC.

This logic is implemented thanks to two Metallb services:
- ipsec-vip with a virtual IP 10.123.123.200 in the external LAN  10.123.123.0/24. This service must be configured with   "externalTrafficPolicy: Local" to prevent any load balancing in IPSEC, which results in inconsistent synchronization between IPSEC  control and data plane. 
- sctp-server-vip1234 with a VIP 1.2.3.4/32. This VIP is somehow internal to the host and attached to no LAN since it is reachable via IPSEC. Depending on the requirements, this service can be configured with externalTrafficPolicy set to "Local or Cluster".

The following diagram summarizes this setup:

```                                          
                                                                  
         vm1                   vm2                   vm3          
+-------------------+ +-------------------+ +-------------------+ 
|                   | |                   | |                   | 
| +------+ +------+ | | +------+ +------+ | | +------+ +------+ | 
| | sctp | | sctp | | | | sctp | | sctp | | | | sctp | | sctp | | 
| | pod1 | | pod2 | | | | pod3 | | pod4 | | | | pod5 | | pod6 | | 
| +------+ +------+ | | +------+ +------+ | | +------+ +------+ | 
|    +-----------------------------------------------------+    | 
|    |                   SCTP VIP 1.2.3.4                  |    | 
|    +-----------------------------------------------------+    | 
|                   | |                   | |                   | 
|  +-------------+  | |  +-------------+  | |  +-------------+  | 
|  |  ipsec ds   |  | |  |  ipsec ds   |  | |  |  ipsec ds   |  | 
|  | hostnetwork |  | |  | hostnetwork |  | |  | hostnetwork |  | 
|  +-------------+  | |  +-------------+  | |  +-------------+  | 
+---------|---------+ +----------|--------+ +---------|---------+ 
  ens3.100|.1            ens3.100|            ens3.100|           
          |                      |                    |           
          |                      |                    |           
     +----------------------------------------------------+       
     |               IPSEC VIP 10.123.123.200             |       
     +----------------------------------------------------+       
          |                      |                    |           
          ---------------------------------------------           
                        vlan external network                     
                           10.123.123.0/24                        
                                 |                                
                                 |.4                              
                +---------------------------------+               
                |                                 |               
                |  SCTP over IPSEC:               |               
                |                                 |               
                |  SCTP SRC=5.6.7.8/32            |               
                |  SCTP DEST=1.2.3.4/32           |               
                |                                 |               
                |  IPSEC SRC=10.123.123.4         |               
                |  IPSEC DEST=10.123.123.200      |               
                |                                 |               
                +---------------------------------+               
                              vm-ext                              
                                                         
```

In the Kubernetes Cluster (vm1), launch the sctp service and deployment and the IPSEC daemonset:
```
kubectl apply -f https://raw.githubusercontent.com/robric/multipass-3-node-k8s/main/source/sctp-mlb-vip1234.yaml
kubectl apply -f https://raw.githubusercontent.com/robric/multipass-3-node-k8s/main/source/strongswan-daemonset.yaml
```

Here is what we get:

```
ubuntu@vm1:~$ kubectl get svc -o wide
NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                        AGE   SELECTOR
ipsec-vip             LoadBalancer   10.43.149.137   10.123.123.200   500:31815/UDP,4500:30285/UDP   40h   app=strongswan
kubernetes            ClusterIP      10.43.0.1       <none>           443/TCP                        44h   <none>
sctp-server-vip1234   LoadBalancer   10.43.212.209   1.2.3.4          10000:30711/SCTP               15h   app=sctp-server-ipsec
ubuntu@vm1:~$ kubectl get pods -o wide
NAME                                 READY   STATUS    RESTARTS   AGE   IP             NODE   NOMINATED NODE   READINESS GATES
ipsec-ds-5lvxc                       1/1     Running   0          21h   10.65.94.22    vm2    <none>           <none>
ipsec-ds-qf8r8                       1/1     Running   0          21h   10.65.94.156   vm3    <none>           <none>
ipsec-ds-ql2nz                       1/1     Running   0          21h   10.65.94.121   vm1    <none>           <none>
sctp-server-ipsec-78c66f958b-4hl9r   1/1     Running   0          15h   10.42.0.38     vm1    <none>           <none>
sctp-server-ipsec-78c66f958b-92zz9   1/1     Running   0          15h   10.42.1.34     vm2    <none>           <none>
sctp-server-ipsec-78c66f958b-cjk4k   1/1     Running   0          15h   10.42.2.33     vm3    <none>           <none>
sctp-server-ipsec-78c66f958b-lppdm   1/1     Running   0          15h   10.42.2.34     vm3    <none>           <none>
sctp-server-ipsec-78c66f958b-qm2w9   1/1     Running   0          15h   10.42.0.39     vm1    <none>           <none>
sctp-server-ipsec-78c66f958b-wlwsg   1/1     Running   0          15h   10.42.1.33     vm2    <none>           <none>
ubuntu@vm1:~$ 
```

Next, to test connectivity of IPSEC/SCTP, use the VM (vm-ext) with the following manifest (it will deploy an IPSEC client with tunnel destination=10.123.123.200 -IPSEC VIP- and SA 5.6.7.8-1.2.3.4):

```
kubectl apply -f https://raw.githubusercontent.com/robric/multipass-3-node-k8s/main/source/strongswan-client.yaml
```

##### Test IPSEC-SCTP-1: externalTrafficPolicy: Cluster == FAILED

Although, things seems to work, since the client ultimately get an answer thanks to restransmissions, we're actually having packet drops: **[RESULT = FAILED]**:

```
#
# This is the result externalTrafficPolicy: Cluster
#

#
# SCTP is actually broken, reaching the VIP 1.2.3.4 (sctp-server-vip1234) 
# However, there are some issues since return trafic to local pods is lost !!!! 
#

ubuntu@vm-ext:~$ sctp_test -H 5.6.7.8  -h 1.2.3.4 -p 10000 -s 
remote:addr=1.2.3.4, port=webmin, family=2
local:addr=5.6.7.8, port=0, family=2
seed = 1720175250

Starting tests...
        socket(SOCK_SEQPACKET, IPPROTO_SCTP)  ->  sk=3
        bind(sk=3, [a:5.6.7.8,p:0])  --  attempt 1/10
Client: Sending packets.(1/10)
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=880700277
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=1633539134
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=1373293174
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=410525908
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=682680410
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=1349506881
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=871858936
[...] 
#
# When running multiple trials, we can see that trafic if sent to any pod the cluster.
# 

# 1/ Find which server owns the VIP 10.123.123.200 by inspecting mac address:

ubuntu@vm-ext:~$ arp -na | grep 123.200
? (10.123.123.200) at 52:54:00:fd:51:52 [ether] on ens3.100
ubuntu@vm-ext:~$ arp -na | grep 52:54:00:fd:51:52
? (10.123.123.200) at 52:54:00:fd:51:52 [ether] on ens3.100
? (10.123.123.1) at 52:54:00:fd:51:52 [ether] on ens3.100  <----- That's vm1

#
# 2/ Connect to vm1 and run conntrack (sudo apt install conntrack) to inspect NAT contexts and check the src 
# of returning traffic (10.42.x.y). 
#
# We can see that the tests are unresponsive when the target pod 
# is running on vm1. These are actually packet drops (see next section for more traces). 
#

#
# This is the result externalTrafficPolicy: Cluster
#

ubuntu@vm1:~$ sudo conntrack -E -p sctp -e NEW
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=34322 dport=10000 [UNREPLIED] src=10.42.2.34 dst=10.42.0.0 sport=9999 dport=28758
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=34863 dport=10000 [UNREPLIED] src=10.42.0.39 dst=10.42.0.1 sport=9999 dport=24858
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=34863 dport=10000 [UNREPLIED] src=10.42.2.33 dst=10.42.0.0 sport=9999 dport=16351
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=58650 dport=10000 [UNREPLIED] src=10.42.1.34 dst=10.42.0.0 sport=9999 dport=27008
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=58028 dport=10000 [UNREPLIED] src=10.42.0.39 dst=10.42.0.1 sport=9999 dport=32527
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=58028 dport=10000 [UNREPLIED] src=10.42.1.34 dst=10.42.0.0 sport=9999 dport=14823
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=44320 dport=10000 [UNREPLIED] src=10.42.1.34 dst=10.42.0.0 sport=9999 dport=31059
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=53050 dport=10000 [UNREPLIED] src=10.42.0.39 dst=10.42.0.1 sport=9999 dport=33676
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=53050 dport=10000 [UNREPLIED] src=10.42.2.34 dst=10.42.0.0 sport=9999 dport=17806
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=49258 dport=10000 [UNREPLIED] src=10.42.2.34 dst=10.42.0.0 sport=9999 dport=44852
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=51474 dport=10000 [UNREPLIED] src=10.42.2.34 dst=10.42.0.0 sport=9999 dport=39112
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=32998 dport=10000 [UNREPLIED] src=10.42.2.33 dst=10.42.0.0 sport=9999 dport=38159
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=38233 dport=10000 [UNREPLIED] src=10.42.0.38 dst=10.42.0.1 sport=9999 dport=12800
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=38233 dport=10000 [UNREPLIED] src=10.42.1.34 dst=10.42.0.0 sport=9999 dport=39258

```
After investigations, we can see that
- E/W packets from vm1 to vm2/vm3 pods are OK
- local packets to VM1 pods are KO (dropped).

##### Test IPSEC-SCTP-2: externalTrafficPolicy: Local == FAILED

Now let's toggle externalTrafficPolicy to "Local" (kubectl edit ...). As we can expect from the previous test, all packets are dropped.

```
#
# Use Kubectl edit svc sctp-server-vip1234 to toggle  
#  externalTrafficPolicy: Cluster -> Local
#


#
#        RESULT ==== It does not work at all !!!
# 

ubuntu@vm-ext:~$ sctp_test -H 5.6.7.8  -h 1.2.3.4 -p 10000 -s
remote:addr=1.2.3.4, port=webmin, family=2
local:addr=5.6.7.8, port=0, family=2
seed = 1720195042

Starting tests...
        socket(SOCK_SEQPACKET, IPPROTO_SCTP)  ->  sk=3
        bind(sk=3, [a:5.6.7.8,p:0])  --  attempt 1/10
Client: Sending packets.(1/10)
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=2055574450
^C
ubuntu@vm-ext:~$ 
#
#[...] NO RESPONSE
#

#
# NAT context are toward local pods (.40 and .41 in this trial):
#
ubuntu@vm1:~$ sudo conntrack -E -p sctp -e NEW
[NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=51396 dport=10000 [UNREPLIED] src=10.42.0.40 dst=5.6.7.8 sport=9999 dport=51396
[NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=51396 dport=10000 [UNREPLIED] src=10.42.0.41 dst=5.6.7.8 sport=9999 dport=51396
[NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=51396 dport=10000 [UNREPLIED] src=10.42.0.41 dst=5.6.7.8 sport=9999 dport=51396

Check pods .40 and .41.
ubuntu@vm1:~$ kubectl  get pods -o wide |grep vm1
[...]
sctp-server-ipsec-78c66f958b-2w62z   1/1     Running   0          28m    10.42.0.41     vm1    <none>           <none>
sctp-server-ipsec-78c66f958b-cwsf5   1/1     Running   0          28m    10.42.0.40     vm1    <none>           <none>
ubuntu@vm1:~$ 

#
# If we inspect at cni0 interface (flannel) where pods are living we see
# that SCTP responses are sent back [INIT ACK].
# 

ubuntu@vm1:~$ sudo tcpdump -vni cni0 "sctp"
tcpdump: listening on cni0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
09:01:15.457601 IP (tos 0x2,ECT(0), ttl 63, id 0, offset 0, flags [DF], proto SCTP (132), length 68)
    5.6.7.8.46020 > 10.42.0.40.9999: sctp (1) [INIT] [init tag: 2450773484] [rwnd: 106496] [OS: 10] [MIS: 65535] [init TSN: 1691014945] 
09:01:15.457960 IP (tos 0x2,ECT(0), ttl 64, id 0, offset 0, flags [DF], proto SCTP (132), length 292)
    10.42.0.40.9999 > 5.6.7.8.46020: sctp (1) [INIT ACK] [init tag: 2794367529] [rwnd: 106496] [OS: 10] [MIS: 10] [init TSN: 3101944263] 
09:01:18.522178 IP (tos 0x2,ECT(0), ttl 63, id 0, offset 0, flags [DF], proto SCTP (132), length 68)
 [...]

#
# But packets never reach properly the ens3.100 interface as if the IPSEC policy was ignored. 
# Only [INIT] packets - two attemps here - 
#

ubuntu@vm1:~$ sudo tcpdump -evni ens3.100 "sctp"
tcpdump: listening on ens3.100, link-type EN10MB (Ethernet), snapshot length 262144 bytes
09:03:23.888239 52:54:00:96:9d:ef > 52:54:00:fd:51:52, ethertype IPv4 (0x0800), length 82: (tos 0x2,ECT(0), ttl 64, id 0, offset 0, flags [DF], proto SCTP (132), length 68)
    5.6.7.8.46020 > 1.2.3.4.10000: sctp (1) [INIT] [init tag: 2714382025] [rwnd: 106496] [OS: 10] [MIS: 65535] [init TSN: 3715017747] 
09:03:27.033937 52:54:00:96:9d:ef > 52:54:00:fd:51:52, ethertype IPv4 (0x0800), length 82: (tos 0x2,ECT(0), ttl 64, id 0, offset 0, flags [DF], proto SCTP (132), length 68)
    5.6.7.8.57282 > 1.2.3.4.10000: sctp (1) [INIT] [init tag: 2714382025] [rwnd: 106496] [OS: 10] [MIS: 65535] [init TSN: 3715017747] 


#
# It looks like we're having some conflicts between IPSEC policies and Netfilter.
#
# Let's just "hack and see" and see if there is a quick win.
#

ubuntu@vm1:~$ sudo sysctl -w net.bridge.bridge-nf-call-iptables=0
net.bridge.bridge-nf-call-iptables = 0
ubuntu@vm1:~$ 

#
#   ===> no luck that did not help (observing packet drops)
#

```
##### Test IPSEC-SCTP-3: use of Hostnetwork with externalTrafficPolicy: Local and Cluster == PASS / PASS

If we deploy sctp server pods in the hostnetwork, we're having a much simpler datapath with no interaction with the CNI.
Either update previous deployment or delete current/apply the following manifest. It will deploy the sctp servers in the hostnetwork with "externalTrafficPolicy: Local".

```
kubectl apply -f https://raw.githubusercontent.com/robric/multipass-3-node-k8s/main/source/sctp-mlb-svc-vip1234-hostnetwork-local.yaml
```

As we can expect, only 1 pod per node can be deployed since there is conflict with the nodeport. So this option does not allow scale-out.

```
ubuntu@vm1:~$ kubectl get pods --selector app=sctp-server-ipsec -o wide
NAME                                 READY   STATUS    RESTARTS   AGE    IP             NODE     NOMINATED NODE   READINESS GATES
sctp-server-ipsec-7697554b65-2z6wm   0/1     Pending   0          110s   <none>         <none>   <none>           <none>
sctp-server-ipsec-7697554b65-6lwx9   1/1     Running   0          110s   10.65.94.22    vm2      <none>           <none>
sctp-server-ipsec-7697554b65-7mpj6   0/1     Pending   0          110s   <none>         <none>   <none>           <none>
sctp-server-ipsec-7697554b65-7xbnb   0/1     Pending   0          110s   <none>         <none>   <none>           <none>
sctp-server-ipsec-7697554b65-r2f5g   1/1     Running   0          110s   10.65.94.156   vm3      <none>           <none>
sctp-server-ipsec-7697554b65-v8slk   1/1     Running   0          110s   10.65.94.121   vm1      <none>           <none>
ubuntu@vm1:~$ 
```

A test from the external SCTP/IPSEC client confirms that this works flawlessly.

```
ubuntu@vm-ext:~$ sctp_test -H 5.6.7.8  -h 1.2.3.4 -p 10000 -s
remote:addr=1.2.3.4, port=webmin, family=2
local:addr=5.6.7.8, port=0, family=2
seed = 1720861856

Starting tests...
        socket(SOCK_SEQPACKET, IPPROTO_SCTP)  ->  sk=3
        bind(sk=3, [a:5.6.7.8,p:0])  --  attempt 1/10
Client: Sending packets.(1/10)
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=1227113623
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=67657223
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=48650977
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=780993105
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=253104860
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=355882517
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=1128917297
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=1206479562
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=56642195
        sendmsg(sk=3, assoc=0)    1 bytes.
          SNDRCV(stream=0 flags=0x1 ppid=1180223014```
[...]
```

```
#
#   conntrack traces confirms that "externalTrafficPolicy: Local" is properly honored:
#             - IPSEC VIP master (here vm2) forwards trafic to local nodeport 10.65.94.22 (see kubectl get pods -o wide above).

    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=44662 dport=10000 [UNREPLIED] src=10.65.94.22 dst=5.6.7.8 sport=9999 dport=44662
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=41286 dport=10000 [UNREPLIED] src=10.65.94.22 dst=5.6.7.8 sport=9999 dport=41286
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=60827 dport=10000 [UNREPLIED] src=10.65.94.22 dst=5.6.7.8 sport=9999 dport=60827
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=60073 dport=10000 [UNREPLIED] src=10.65.94.22 dst=5.6.7.8 sport=9999 dport=60073
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=38883 dport=10000 [UNREPLIED] src=10.65.94.22 dst=5.6.7.8 sport=9999 dport=38883
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=51788 dport=10000 [UNREPLIED] src=10.65.94.22 dst=5.6.7.8 sport=9999 dport=51788
```

We can change the service configuration to externalTrafficPolicy: Cluster to verify if load balancing works properly.

```

#
# Run multiple tests with from the client side:
#

ubuntu@vm-ext:~$ sctp_test -H 5.6.7.8  -h 1.2.3.4 -p 10000 -s
remote:addr=1.2.3.4, port=webmin, family=2
local:addr=5.6.7.8, port=0, family=2
seed = 1720862539

Starting tests...
        socket(SOCK_SEQPACKET, IPPROTO_SCTP)  ->  sk=3
        bind(sk=3, [a:5.6.7.8,p:0])  --  attempt 1/10
Client: Sending packets.(1/10)
[...]

#
# Check the result of NAT translation in conntrack and look at the src= for nodeports IP (target SCTP server hooks).
# Note also the enforcement of SNAT with the IPSEC VIP owner nodeport IP (dst=10.65.94.22)
#

ubuntu@vm2:~$ sudo conntrack -E -p sctp -e NEW
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=40375 dport=10000 [UNREPLIED] src=10.65.94.22 dst=5.6.7.8 sport=9999 dport=40375
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=50512 dport=10000 [UNREPLIED] src=10.65.94.22 dst=5.6.7.8 sport=9999 dport=50512
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=35412 dport=10000 [UNREPLIED] src=10.65.94.121 dst=10.65.94.22 sport=9999 dport=50422
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=46142 dport=10000 [UNREPLIED] src=10.65.94.121 dst=10.65.94.22 sport=9999 dport=8137
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=46735 dport=10000 [UNREPLIED] src=10.65.94.121 dst=10.65.94.22 sport=9999 dport=8345
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=60532 dport=10000 [UNREPLIED] src=10.65.94.156 dst=10.65.94.22 sport=9999 dport=43488
    [NEW] sctp     132 10 CLOSED src=5.6.7.8 dst=1.2.3.4 sport=52803 dport=10000 [UNREPLIED] src=10.65.94.156 dst=10.65.94.22 sport=9999 dport=1521
```

##### Test Results Summary

The below list summarizes the test results:
-  HostNetwork: False + externalTrafficPolicy: Local   ===> FAILED
-  HostNetwork: False + externalTrafficPolicy: Cluster ===> FAILED
-  HostNetwork: True + externalTrafficPolicy: Local    ===> PASS
-  HostNetwork: True + externalTrafficPolicy: Cluster  ===> PASS

### Troubleshooting

Checks the logs of the speaker to track ownership of VIP. This is actually a daemonset that runs in the hostnetwork.

```
ubuntu@vm1:~$ kubectl get pods -o wide -n metallb-system 
NAME                                      READY   STATUS    RESTARTS   AGE    IP             NODE   NOMINATED NODE   READINESS GATES
frr-k8s-webhook-server-7d94b7b8d5-8pgd7   1/1     Running   0          7d4h   10.42.1.7      vm2    <none>           <none>
frr-k8s-daemon-t68fr                      6/6     Running   0          7d4h   10.65.94.199   vm2    <none>           <none>
controller-5f4fc66d9d-4j4h5               1/1     Running   0          7d4h   10.42.1.8      vm2    <none>           <none>
frr-k8s-daemon-q695g                      6/6     Running   0          7d4h   10.65.94.238   vm1    <none>           <none>
speaker-gq27n                             1/1     Running   0          7d4h   10.65.94.238   vm1    <none>           <none>
speaker-lklxw                             1/1     Running   0          7d4h   10.65.94.199   vm2    <none>           <none>
frr-k8s-daemon-h7rh6                      6/6     Running   0          7d4h   10.65.94.95    vm3    <none>           <none>
speaker-n9cbf                             1/1     Running   0          7d4h   10.65.94.95    vm3    <none>           <none>
ubuntu@vm1:~$ 

#
# This is where you see how ARP are generated 
#

ubuntu@vm1:~$ kubectl logs -n metallb-system speaker-gq27n  | grep -i arp
{"caller":"announcer.go:126","event":"createARPResponder","interface":"ens3","level":"info","msg":"created ARP responder for interface","ts":"2024-06-12T12:31:56Z"}
{"caller":"announcer.go:126","event":"createARPResponder","interface":"ens3.100","level":"info","msg":"created ARP responder for interface","ts":"2024-06-12T12:31:56Z"}

```

### Issues

After deletion/recreation, the metallb svc is easily stuck in pending state.

'''
ubuntu@vm1:~$ kubectl  get svc
NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                        AGE
ipsec-vip             LoadBalancer   10.43.158.165   <pending>     500:32572/UDP,4500:31479/UDP   75s
kubernetes            ClusterIP      10.43.0.1       <none>        443/TCP                        12d
sctp-server-vip1234   LoadBalancer   10.43.212.209   1.2.3.4       10000:30711/SCTP               10d
ubuntu@vm1:~$ 
'''

I managed to have things working by 
- deleting svc, metallb l2advertisment and ipaddresspool 
- flusing arp entry for VIP: 10.123.123.200 

```
ubuntu@vm2:~$ sudo arp -d 10.123.123.200
ubuntu@vm2:~$ 
```