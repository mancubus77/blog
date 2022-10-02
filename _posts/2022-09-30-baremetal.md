OpenShift Baremetal Ingress for Telco use cases
===============================================

Overview
========

Kubernetes is great tool and it has got a niche in IT segment. Almost everyone talks about containers, pods, services, ingresses and can find a use case what can be solved with Kubernetes. Nevertheless, it’s relatively new in Telecom segment, but eventually it will be in same row with well industry recognized Platform-As-Service. Keeping in mind SDx (Software Defined Everything) it will happen very soon. Vendors, Open Source communities and Independent Software Vendors introduce concept of CNF (Cloud Native Functions) what can run in containers instead of VMs to replace VNFs and Baremetal installations. Kubernetes community is working hard to make it happen and develop more and more functions to make the platform efficient, performance and reliable as VIM (like Open Stack or VMWare).
I was working on cloud native Telemetry stack for even driven telemetry and faced a couple of challenges with OpenShift, so let’s speak about them and how to solve them. In particularly let’s speak about how traffic gets into the cluster from network elements. Kubernetes can fairly easy manage and manipulate HTTP with routes and ingress. Let’s talk about L4 TCP/UDP traffic what is common in telecom.

Why would we need not HTTP traffic?
-----------------------------------

Before start looking on details let’s understand why do we need get into cluster via non-HTTP traffic? We have many use cases in Telco and the simplest one is network monitoring. If we want to build cloud native high available system to measure a health of the network we need to allow access to the cluster for a bunch of non-HTTP protocols like SNMP Traps (over UDP), BGP (over TCP) for BGP Monitoring and topology tools, Telemetry (over TCP or gRPC/HTTP2) for event driven telemetry and of course `<put your favorite use case here>`.

How to traffic can get to cluster
=================================

If you are familiar with Kubernetes you should know a few methods how traffic getting into the cluster. Let’s put aside Ingress Controllers and speak about Services and it types. You may stop me here and object, but bear with for a while. We intentionally do not considering Ingress, because it’s something behind load-balancer and our discussion is around getting into the cluster in HA manner.

![](https://miro.medium.com/max/1400/1*VAbORTUTFihQcAEAGoGleQ.png)Entry points

As you see from picture above, Kubernetes will take care about traffic when it will reach ingress, but it’s not managing Global Server Load Balancer (unless you are on cloud) Meanwhile Kubernetes service allows to expose Pods port to the cluster or outside world. So what do we have here?

ClusterIP
---------

![](https://miro.medium.com/max/992/1*_Bgj-AVaOl2mWGxWz82fZQ.png)ClusterIP

This is a basic method to expose pod’s port and IP. Pod’s IP (assigned by OVN) and PORT will be available within cluster via FQDN (Fully Qualified Domain Name) e.g. `cluster-kafka-brokers.kafka.svc.cluster.local:9092` _or_ `<**service-name**>.<**namespace-name**>.svc.cluster.local:<**port**>` or simple `<pod-ip>:<port>` Within same namespace it can be called `cluster-kafka-brokers` _or_ `<service_name>`_._ It is not possible connect to pods from outside of the cluster. This useful method to link service together but not helpful for our Telecom Application.

NodePort
--------

Node port is getting us closer to connection with our cluster.

![](https://miro.medium.com/max/1400/1*pjJXU4vmnskFep2Jdcwxzw.png)

Now we have IP and Port to establish TCP/UDP connection! But we need to be aware about a couple of caveats of Kubernetes design.
NodeIP is iptables entry what maps PORT to Service, therefore:
Ports can be only within range `30,000~32,767` (by Kubernetes Design)
Port will be exposed to all worker nodes in the cluster, so to access it you need to know worker IP address. In case you have baremetal installation of OpenShift, presumably you have HAProxy/NGINX or something more cool like F5 in front of your cluster. This entity forwards traffic according Red Hat’s [recommendations](https://www.openshift.com/blog/haproxy-highly-available-keepalived) and to get to any of your nodes you might need to access your cluster. First issue that you might notice that recommended settings for HAProxy doesn’t forward custom ports (`30,000~32,767`), therefore even you try to resolve your cluster via `**WhatEvenYouWantPutHere**.apps.**<YOUR-CLUSTER-NAME>**.**<YOUR-DOMAIN-NAME>**:30000`  it won’t go through default HAProxy configuration. So if you prefer to use NodeIP method you might need to update your HAProxy configuration for each application or even make mapping of `<YOU LIKE PORT>` to `<K8s LIKES PORT>` It’s good, but not what we are after

LoadBalancer
------------

How is about to let Kubernetes update port mappings and nodes automatically and keep it actual despite application lifecycle? Sounds good right? You can get it, but this is available only for cloud installation (what we Telco do not have because we need to be OnPrem (on premises). For sure you can use various CIS (Custom Ingress Services) like [F5 BIG-IP](https://clouddocs.f5.com/containers/v2/kubernetes/) platform it will do same what AWS/AZURE/GCP do and it will be integrated with your infra and GSLB/SLB. OK, but we are telco and that means we want cool stuff with more control and less vendor lock-in (Sorry F5). So far I would like to explore 2 projects within this post, unfortunately both of them not supported by Red Hat, but I believe it’s matter of time when it will be part of Red Hat OpenShift.

Keepalived Operator
-------------------

[Keepalived Operator](https://github.com/redhat-cop/keepalived-operator) is Kubernetes operator of well know and acknowledged application within IT industry application called [Keepalived](https://www.keepalived.org). Long story short, you getting one or several VIP (Virtual IP addresses or Floating IP) which can be assigned to elected node via VRRP (Virtual Router Redundancy Protocol). In case main node node become unavailable, VRRP detects it and reallocate VIP to another active node. VRRP is essential between workers as they will be negotiating who should carry IP. So, in case we would like to spin up BGP Client on your Kubernetes cluster what will accept connections from routers you might need to do following steps:

*   Define a pool of IP addresses (sure thing within same CIDR of your worker nodes)
*   Deploy “keepalived operator” on your OpenShift/Kuberneres cluster.
*   Deploy BGP Speaker (ExaBGP, goBGP, Bird, Quagga … on K8S cluster) and expose service with “[Load Balancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)” type and “[External IP](https://kubernetes.io/docs/concepts/services-networking/service/#external-ips)” address.
*   Operator will do other things for you.
*   Setup BGP session with your cluster
*   Enjoy BGP full view copy on your Kubernetes cluster (Why? May be to create another looking glass or monitoring anomaly of your AS or load them in Graph Database)

**Cool things about Keepalived operator:**

*   It’s Kubernetes operator, therefore it’s taking care about application lifecycle
*   It can be assigned to particular node (Hello VRF!)
*   It supports cloud native monitoring
*   It can co-exist with OpenShift native services (be aware about support)

Please be noted that keepalived operator doesn’t provide pure load balancing capabilities like keepalived virtual server, only High Availability, therefore all traffic to service will go through one node.

MetalLB
-------

[MetalLB](https://metallb.universe.tf/) is new and very promising project of cloud native load balancing with support of VIP (Virtual IP) and BGP ECMP (💜💘♥). This beast supports 2 configuration methods Layer2 and BGP. Please be noted project is “Beta”.

**Layer2**
This method uses same concept with Keepalived, with a couple of differences in realisation. For example MetalLB L2 uses [memerlist](https://github.com/hashicorp/memberlist) based on [gossip](https://en.wikipedia.org/wiki/Gossip_protocol) not VRRP like keepalived, but overall concept is nearly the same.

**BGP
**Now we are talking! BGP is well know and beloved protocol for Telco and everyone respect it. If you are not familiar with BGP, have a look [here](https://www.kentik.com/blog/bgp-routing-tutorial-series-part-1/). So MetalLB can establish BGP session with BGP router and advertise virtual IP to your network.

![](https://miro.medium.com/max/1004/1*eOHDodfkr8Y9JAFq9TAYwQ.png)ECMP with MetalLB

Also MetalLB supports ECMP, therefore traffic will be speaded across all BGP Speakers in cluster.

![](https://miro.medium.com/max/1400/1*eF0xi5-SXAiLjqFkzGoOng.png)source: [https://metallb.universe.tf/configuration/](https://metallb.universe.tf/configuration/)

**Cool things about MetalLB:**

*   Two modes L2 and BGP
*   Supports Ingress
*   Pin L2/BGP to worker nodes
*   Compatible with CNI (Container Network Interface)
*   Load Balancing with ECMP
*   Almost unlimited number of L2 routers (VRRP has limit of 255)

Conclusion
----------

With help of methods above you easily define an IP address or FQDN with one of floatingIPs in a pool what can be used by all your network elements to send SNMP Traps or configured as destination address for Event Driven Telemetry. MetalLB or Keepalived take care about IP availability and Kubernetes make sure that service is resilient and available

OpenShift provides new capabilities for Telco and Open Source community is moving forward to make it happen. OpenShift and Kubernetes are dominating in clouds, but in next few years it will be another standard for Telco Applications.
In terms of load-balancing MetalLB looks very promising from my point of view and allow to make real cool stuff, meanwhile Keepalived is mature and acknowledged in the community.
