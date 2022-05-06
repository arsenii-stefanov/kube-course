## How K8s Service (ClusterIP) works

### What is ClusterIP?

ClusterIP is a set of iptables rules that forward packets to a single entry point (Service ClusterIP's IP address) to a group of Pods with the same set of labels (i.e. Pods that match Service's selectors).

### Why is it needed?

If a Pod serves HTTP/TCP/UDP requests and has several replicas for high availability and/or high load, those Pods likely have a load balancer in front of them. The load balancer proxies incoming traffic to one or several back-ends which could be IPs (or hostnames) of nodes or containers where the target application is running. Pods are stateless, they can be terminated and re-created due to numerous reasons, they can be scaled in or out, their IP addresses are dynamic. K8s Service that creates an entrypoint with a relatively static IP (even if Service is deleted and re-created, a corresponding Ingress resource which is responsible for linking the Service and a load balancer would ensure the load balancer has an up-to-date back-end configuration) that serves as a back-end for our load balancer and an entrypoint for our application. Nevertheless, some load balancer controllers prefer to add Pods' IPs directly to load balancers instead of using Service's IP, for example AWS Load Balancer Controller; note that this requires your K8s cluster to be so called 'VPC Native' Pods to be accessible within the VPC, not only within the K8s cluster; on Google Cloud, you would need to create a cluster with the 'IP Alias' option enable, while on AWS you would need to deploy a Daemonset called AWS VPC CNI (it creates a network interface for each Pod).

### How does it work under the hood?

Each K8s node has a Pod of an application called 'kube-proxy' which is responsible for managing iptables rules on the K8s node. Let's say we have a Deployment (an application with an HTTP server) with 4 Pods (replicas), each Pod is located on a different node. When we create an object of kind Service (type=ClusterIP), 'kube-proxy' adds a set of iptables fules on each K8s worker node (which is why you can access your Service and Pod by their IPs from any node in the cluster). When a client sends a request to the load balancer, the LB proxies it to its back-end (simply an <IP>:<PORT>) which is a K8s object of type Service.

Basically, when a TCP/UDP packet arrives to a node (any node), it matches some iptables rules and is forwarded to its final destination, one of 4 Pods with our application. A Pod is selected randomly.

Let's take a closer look.

* Find the IP address of some K8s Service and IP addresses of Pods located 'behind' that Service:

```
$ kubectl -n stateful-apps get svc
NAME     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
my-app   ClusterIP   10.104.5.205   <none>        8080/TCP   5h56m

$ kubectl -n stateful-apps describe svc my-app
Name:              my-app
Namespace:         stateful-apps
Labels:            app=my-app
Annotations:       <none>
Selector:          app=my-app,type=web-app
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.104.5.205
IPs:               10.104.5.205
Port:              http-svc  8080/TCP
TargetPort:        app-http/TCP
Endpoints:         172.17.0.2:80,172.17.0.5:80,172.17.0.6:80 + 1 more...
Session Affinity:  None
Events:            <none>

$ kubectl -n stateful-apps get pod -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
my-app-7f569f66d9-6dkwl   2/2     Running   0          4h39m   172.17.0.7   minikube   <none>           <none>
my-app-7f569f66d9-dfrdh   2/2     Running   0          4h39m   172.17.0.2   minikube   <none>           <none>
my-app-7f569f66d9-qtc2q   2/2     Running   0          4h39m   172.17.0.5   minikube   <none>           <none>
my-app-7f569f66d9-ztt65   2/2     Running   0          4h39m   172.17.0.6   minikube   <none>           <none>
```

* SSH into the minikube node (think of it as a regular K8s worker node) / Alternatively, if you are working with a real cluster, you can 'kubectl exec' into a 'kube-proxy' pod and use the same commands:

```
$ minikube ssh -n minikube
```

* We will jump to the point where a TCP packet has already reached the KUBE-SERVICES rule. Check the rule for the IP of our Service (ClusterIP) in the 'KUBE-SERVICES' chain of the 'NAT' table:

```
$ docker@minikube:~$ sudo iptables -t nat -L KUBE-SERVICES -v -n | grep 10.104.5.205
    0     0 KUBE-SVC-Q7IJ732UBZY4AJKA  tcp  --  *      *       0.0.0.0/0            10.104.5.205         /* stateful-apps/my-app:http-svc cluster IP */ tcp dpt:8080
```

This means that if a packet is sent to 10.104.5.205:8080, it will first match the 'KUBE-SVC-Q7IJ732UBZY4AJKA' rule. '10.104.5.205:8080' can be used as a back-end for our load balancer which it proxies all incoming requests to.

* Let's see what's in that rule:

```
$ docker@minikube:~$ sudo iptables -t nat -L KUBE-SVC-Q7IJ732UBZY4AJKA -v
Chain KUBE-SVC-Q7IJ732UBZY4AJKA (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  tcp  --  any    any    !10.244.0.0/16        10.104.5.205         /* stateful-apps/my-app:http-svc cluster IP */ tcp dpt:http-alt
    0     0 KUBE-SEP-TWZJG7ZWZYI5EC7J  all  --  any    any     anywhere             anywhere             /* stateful-apps/my-app:http-svc */ statistic mode random probability 0.25000000000
    0     0 KUBE-SEP-LCP7KINQA756A5JS  all  --  any    any     anywhere             anywhere             /* stateful-apps/my-app:http-svc */ statistic mode random probability 0.33333333349
    0     0 KUBE-SEP-XCVGJ5SWPGOUASPD  all  --  any    any     anywhere             anywhere             /* stateful-apps/my-app:http-svc */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-56377YKM7VKGGV6C  all  --  any    any     anywhere             anywhere             /* stateful-apps/my-app:http-svc */
```

There're 2 types of rules in the rule above: the first type is KUBE-MARK-MASQ, which adds a tag to the packet that came from an IP different from the Pods CIDR (10.244.0.0/16), and the second type is a rule for each Pod that matches the Service's selectors:

```
$ docker@minikube:~$ sudo iptables -t nat -L KUBE-MARK-MASQ -v -n
Chain KUBE-MARK-MASQ (21 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK or 0x4000
```

* Below is a rule (one of 4) that routes the packet to a target Pod (one of 4), namely 'my-app-7f569f66d9-dfrdh' with the IP 172.17.0.2 and port 80:

```
$ docker@minikube:~$ sudo iptables -t nat -L KUBE-SEP-TWZJG7ZWZYI5EC7J -v -n
Chain KUBE-SEP-TWZJG7ZWZYI5EC7J (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       172.17.0.2           0.0.0.0/0            /* stateful-apps/my-app:http-svc */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* stateful-apps/my-app:http-svc */ tcp to:172.17.0.2:80
```

* Check out the rest 3 rules to see that they forward packets to the 3 other Pods:

```
$ docker@minikube:~$ sudo iptables -t nat -L KUBE-SEP-LCP7KINQA756A5JS -v -n
Chain KUBE-SEP-LCP7KINQA756A5JS (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       172.17.0.5           0.0.0.0/0            /* stateful-apps/my-app:http-svc */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* stateful-apps/my-app:http-svc */ tcp to:172.17.0.5:80

$ docker@minikube:~$ sudo iptables -t nat -L KUBE-SEP-XCVGJ5SWPGOUASPD -v -n
Chain KUBE-SEP-XCVGJ5SWPGOUASPD (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       172.17.0.6           0.0.0.0/0            /* stateful-apps/my-app:http-svc */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* stateful-apps/my-app:http-svc */ tcp to:172.17.0.6:80

$ docker@minikube:~$ sudo iptables -t nat -L KUBE-SEP-56377YKM7VKGGV6C -v -n
Chain KUBE-SEP-56377YKM7VKGGV6C (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       172.17.0.7           0.0.0.0/0            /* stateful-apps/my-app:http-svc */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* stateful-apps/my-app:http-svc */ tcp to:172.17.0.7:80
```
