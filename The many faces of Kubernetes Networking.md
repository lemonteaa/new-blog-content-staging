# The many faces of Kubernetes Networking

Kubernetes is an enormous piece of software with an impressively large ecosystem, and its advocates argue that for a wide range of use cases (such as your typical web application once it reaches certain scale), using it instantly gives you a whole bunch of DevOps goodies "for-free". Unfortunately, its largeness is equally matched by its intricacies - Kubernetes (or k8s) is infamous for being a sprawling system, constantly in flux (there are many active developments going on behind the scene), and once you go beyond using `kubectl` and want to dig below the system layer, you'd find many subtle details.

It doesn't help that Networking, by itself, is already a fairly difficult subject: not all software developers have experience in this field, and the primitives/abstractions of networking is varied, very context-depedent, and doesn't compose nicely.

This post is the author's own log of a journey trying to peel back the layers and perhaps, telling a story.

(Note: the beginning sections is deliberately verbose and perhaps "too easy" for some people's taste. If that's the case, please skip the section and go straight into the core. What I thought is that if someone need to read the prerequisite, then his background is not yet ready for advanced materials, and in that case, having a gradual ramp up seems to be a more sensible choice.)

# Networking Prerequisite

This part is long. The spirit of the matter is that there are many networking primitives/techniques available for upper layer in the tech stack to pick and choose for their use. From k8s's perspective, these are all just tools to help realize the goal of automatic management and realization of resilient, powerful systems. If you ever get lost, don't panic, and remember - it is the spirit that matters.

## NAT, iptables and conntrack

If you haven't been sleeping through the network lecture, you probably know the very basic of what NAT (Network Address Translation) is. And yet the details are messy - indeed my guess is that for many layperson, the most common networking problem they encounter in real life is trouble with NAT somewhere (e.g. Playing computer with friends over network). We will need to know a bit more to proceed.

Two trivial points that I used to forget while thinking NAT - remember, IP address is at IP Layer (L3/Layer 3), while port number is a Transport Layer (L4) concept. Yes, L4 is in between L3 and L5. For some reason my brain has a blindspot on L4.

The second point is that on L5, the L4 part of the pakcet would need a port number for both the source and destination. We all know about the reserved port range () are for well known protocol and is a standard, like 80 for http and 443 for https. But consider a home user browsing the internet - what is his source port? Well, pick a random high number port ;). This is also how it is possible to have multiple connections at the same time - you pick a different port number. Guess I'm brain-damaged afterall, which is pretty sad.

Anyway, on to real business.

Turns out that there are different kinds of NAT. For the purpose of this article, we only need to know S-NAT (Source NAT) and D-NAT (Destination NAT).

- S-NAT: The router changes the source address of the packet. The basic example is client computers sitting behind a (gateway) router, wanting to browse the internet (in other words, home computer user). As the client is on a private network on a local subnet (such as the RFC 192.168 or 10.0), if you just send the packet out to the internet, it could reach the receiver, but then the receiver won't be able to send the response and have it routed to you. In the naive version, we just add a routing rule to the router: change the source IP to the router's public IP address, and remember that this port is conceptually "mapped" to that client (i.e. the client's IP + port). Then when the response arrive at the router (since from the outside it would appear as if the router is the source), read from the mapping to translate the destination back to the real recipient in order to route it.
 - Note that it is preferrable to allow also changing the source port, and remember the full port <-> clientIP:clientPort mapping. The reason is because there could be two clients sending from the same port, and if you don't have this flexibility, there would be a port collision.
 - S-NAT is sometimes also called stateful NAT because of the need to remember the mapping, instead of just a fixed routing rule. But even this is not good enough because there actually is a security vulnerability! If Eve peeked at the traffic (or just do a port scan) and knows the port, he could send a packet out of the blue into that port at the router, and the router, in the naive version, would happily route it to the client. This is a big no-no. The right thing to do is that we should only let packets in if they are in respond to something the client have sent. In order to judge this it is necessary to inspect the packets and track the connection's (TCP) protocol state. (More on this later)
- D-NAT: Kind of the opposite. But it is not exactly symmetrical to S-NAT (because networking isn't symmetrical either - remember, we're still on the client-server architecture and not true p2p, at least, not yet). Another name for it is "port-forwarding". The basic example is someone wanting to expose web service on his computer to the outside world in a controlled manner. If you have administrative control on the router, you'd set (or ask it to set) a routing rule: whenever a packet arrives at the router with destination routerIP:publishedport, please rewrite the destination into clientIP:exposedport.


(Sidenote on SNAT home user example: why not just give the client a public IP address? Well the problem is that in IPv4 at least, public IP is a very scarce commodity, and so you'd need to share with other clients in the same local network. Indeed, for home user, you'd appear to the outside world as having an ISP IP. This NAT carried out by the ISP is known as Carrier-Grade NAT (CG-NAT). Note that this case is more complicated in that the clients are inside the ISP network, and allocated a local IP through DHCP so that it is dynamic. To make things worse, the usual home network setup is to have a home router act as "the client" from the ISP's perspective, which itself uses NAT on the home network. First, this make it necessary to use dynamic NAT on the home router. But the real problem is that this make certain tasks difficult to do, and is known as the [double-NAT issue](https://openwrt.org/docs/guide-user/network/switch_router_gateway_and_nat). With IPv6 however addresses would be cheap, so that something that break composability and the end-to-end principle like NAT wouldn't be used in the first place, and rightfully so.)

Phew! That's a lot for the beginning of the prerequisite section! Now we're onto practise. Turns out that Linux has a really powerful networking stack, way beyond what is minimally needed to browse the web - it comes packed with lots of networking tools/libraries. Perhaps the most wellknown is `iptable`. Basically, you can have your Linux act as a router. It is also notorious for having a complicated routing logic.

In particular, it can do various forms of NAT. Like the famous `MASQUARADER`. But that itsn't the end all. Remember the point about needing to know what state in TCP protocol a packet's connection is in? Well, Linux has `conntrack`.

(TODO: netfilter)

## Network Namespace, Virtual Ethernet, and Bridges

To build on top of the last section, it is not just the end user tools. The Linux Kernel itself also have the need of System Admin and Network Engineers in mind. For someone who have just taken a basic OS course, "Linux Networking" may only mean the socket library and how the kernel handle the internal buffers, interface with the NIC (Network Interface Card), etc. But, the kernel actually architectured it with something akin to a plugin system - there are various points where one can extend networking capabilities in a native way.



## L2 Encapsulations - vxLAN and friends

- Main reference: https://vincent.bernat.ch/en/blog/2017-vxlan-linux

TODO: Re-org

Linux also provide supports for ARP tables for virtual devices. To add an entry:

```bash
ip neigh add <IP-ADDR> lladdr <CORRESPOND-MAC-ADDR> dev <DEV>
```

In a similar manner, the FDB (Forwarding Database) for virtual bridge is also configurable:

```bash
bridge fdb add <MAC-ADDR> dev <DEV>
```

To create a VXLAN tunnel in Linux, issue the following command:

```bash
ip link add vxlan100 type vxlan \
   id 100 \
   dstport 4789 \
   local <IP-ADDR> \
   nolearning \
   l2miss \
   l3miss \
   proxy
```

Now here's the trick. Only when the device in an FDB entry is of a VXLAN type, some additional parameters are supported in the entry. Quoting from the mannual:

> The next command line parameters apply only when the specified device DEV is of type VXLAN.
> 
> dst IPADDR
> 
> the IP address of the destination VXLAN tunnel endpoint
> where the Ethernet MAC ADDRESS resides.
>
> src_vni VNI
> 
> the src VNI Network Identifier (or VXLAN Segment ID) this
> entry belongs to. Used only when the vxlan device is in
> external or collect metadata mode. If omitted the value
> specified at vxlan device creation will be used.
>
> vni VNI
> 
> the VXLAN VNI Network Identifier (or VXLAN Segment ID) to
> use to connect to the remote VXLAN tunnel endpoint.  If
> omitted the value specified at vxlan device creation will
> be used.
>
> port PORT
> 
> the UDP destination PORT number to use to connect to the
> remote VXLAN tunnel endpoint.  If omitted the default
> value is used.
>
> via DEVICE
> 
> device name of the outgoing interface for the VXLAN device
> driver to reach the remote VXLAN tunnel endpoint.
>
> nhid NHID
> 
> ecmp nexthop group for the VXLAN device driver to reach
> remote VXLAN tunnel endpoints.

This is how Linux know which destination VTEP to send the encapsulated packet to once the original MAC frame reaches the host's own VTEP.

But there is more. Later, we will need to use this for Flannel, a k8s CNI plugin/addon. The approach it uses is "Unicast with dynamic L3 entry". What does this mean?

We can understand it step by step. Consider the simpler case of "Unicast with static L3 entry". If we know beforehand all endpoints in the combined, virtual, ethernet domain - that is, we know all of their IP + MAC address association - then we can just "hardcode everything". Unpacking, this means that on each host, we prepopulate the ARP table with those mappings that live on other hosts, then the "proxy" option will ensure that **ARP proxy** is enabled, so that these ARP query will be answered correctly. Then, once the packet reach L2, the (again hardcoded) FDB will forward then to the VXLAN, and the additional option will ensure that the packet reappear on the right host to continue the journey.

The dynamic here simply means that we keep updating all the entries needed to make the static case work, as the destinations change their IP address, move around, etc. This is where yet another trick in Linux appears: **netlink sockets**. What may appears to some as a dirty hack, is a key to modern Linux networking by others. At any rate, it is an interesting story.

https://unix.stackexchange.com/questions/93412/difference-between-ifconfig-and-ip-commands

> Just to add some bits to the answer by pilona. Around 2005 a new mechanism for controlling the network stack was introduced - netlink sockets.
> To configure the network interface iproute2 makes use of that full-duplex netlink socket mechanism, while ifconfig relies on an ioctl system call. Here are 2 main papers on motivation behind netlink and usage of rtnetlink.

Anyway, citing the reference:

> Notifications are sent to programs listening to an `AF_NETLINK` socket using the `NETLINK_ROUTE` protocol. This socket needs to be bound to the `RTNLGRP_NEIGH` group.

In short - there should be an agent program on the host, which listen to the relevant sockets. As it receives these messages, it should then query the k8s api server to retrieve the "right answer", and update all the needed entries.

# Kubernetes Prerequisite

## Kubernetes Architecture

## Controller and Synchronisation Loop

## Some important objects

# The k8s Networking Model

There is a certain flexibility in implementing networking for k8s - as long as you fulfill the specified *Networking Model*. In short,

> - all Pods can communicate with all other Pods without using network address translation (NAT).
> - all Nodes can communicate with all Pods without NAT. [^1]
> - the IP that a Pod sees itself as is the same IP that others see it as.

As this article pointed out, the underlying philosophy is that "Every Pod has a unique IP", and that "it is routable from all the other Pods". In other word, conceptually all Pods should behave as if they live on a flat network.

The no NAT requirement is because we would be using NAT for higher level stuff.

# Basic Setup

https://linux-blog.anracom.com/2017/11/14/fun-with-veth-devices-linux-bridges-and-vlans-in-unnamed-linux-network-namespaces-iii/


Let's start with a toy example: single node cluster, multiple Pods inside.

Since one of the point of container is *isolation*, we want the network to be isolated too. Hence we want each Pod to live inside its own Linux Network Namespace:

```bash
ip netns add ns1
ip netns add ns2
```

We then wire then to the root namespace using Linux's **virtual ethernet (veth)**. Note that virtual ethernet always come in pair.

```bash
ip link add veth11 type veth peer name veth10
ip link add veth22 type veth peer name veth20
```

Then we plug it in to the respective network namespace:

```bash
ip link set veth11 netns ns1
ip link set veth10 netns root
ip link set veth22 netns ns2
ip link set veth20 netns root
```

And setup ip addresses for the pods:

```bash
ip addr add 192.168.1.1/24 brd 192.168.1.255 dev veth11
ip addr add 192.168.2.1/24 brd 192.168.2.255 dev veth22
```

To let them talk to each other, we implement a flat network by using a Linux Bridge. It is controlled by the `brctl` command:

```bash
brctl addbr brx
brctl addif brx veth10
brctl addif brx veth20
```

Remember to UP everything: `ip link set <device> up`.

In short, this is a software emulation of the basic networking scenario of IP network over a single Ethernet domain: all computers are connected by a L2 bridge, IP packets go down to L2, reach the bridge, then arrive at the destination and go back up the layer. To be precise, recall some basic networking:

- Linux Bridge is a learning bridge by default: it will broadcast an ethernet frame if it doesn't yet know the MAC address, and will remember where (which interface) the reply comes from. The next time a frame with that MAC as destination is received, it will forward it on that interface only.
- ARP is used to resolve IP address to MAC address. This is handled transparently by the bridge - it treat it just like any other ethernet frame.

> Hint: you can check the MAC table/forwarding database of a linux bridge by the command `brctl showmacs <bridge-name>`

This is all well and good, but what about pods distributed over multiple nodes? Kubernetes implement this via a CNI Plugins, which is the next section.

# CNI Plugins for internode Pod communication

https://medium.com/@jain.sm/flannel-vs-calico-a-battle-of-l2-vs-l3-based-networking-5a30cd0a3ebd


TODO: CIDR policy

## Flannel

Flannel takes a L2 approach by using vxLAN. A virtual ethernet device `flannel0` is added in the node that act as the VTEP: packets sent here will be encapsulated, sent over the tunnel, then unpacked to recover the packet.

The nice thing about this method is that it is conceptually simple: from the Pods' point of view, all Pods live on the same ethernet domain even though they may reside on different nodes. Thus we mostly preserve the flat network picture in the basic setup above.

Now the downside: the implementation is subtle, and vxLAN involves a fair bit of overhead since it is an overlay network.

## Calico

- Main reference: https://docs.projectcalico.org/reference/architecture/data-path

On the other hand, Calico takes a more direct, L3 approach. Imagine turning each of the k8s node into a BGP peer. Each node knows the Pods that are running on them, and so announce a route for each Pod by specifying the node's IP as the next hop. Other nodes receive these information, and update the `iptable` entry correspondingly. Thus an inter-node, Pod-to-Pod packet is natively routed.

This may be confusing at first so here's an example:

TODO

Alas, there's a few subtle point to notes.

First, there's a problem: ARP queries won't work! Why? Because after L3 is done, the packet will go down to L2, and it want to ARP the next hop IP's MAC address. But that is a node IP, and it lives on the L2 network connecting the host, while our packet's still inside the virtual ethernet network inside the node. As ARP is L2, it normally cannot cross ethernet domain.

In Calico, this problem is solved by using an **ARP proxy**. It answers the ARP query on behave of the destination node.

Secondly, the exact mechanism used by Calico is a bit in reverse: Calico install an *agent* called `felix` on each node, and it adds the routes to the iptable during the provisioning of a Pod. After that, the BGP client (e.g. BIRD) notices the entry and then announce it to other nodes.

> BGP sounds scary as it is usually used at the internet core, but it need not be so. Recall that there is a distinction between iBGP vs eBGP - our use case here is more akin to iBGP/internal BGP. The reason for using it is that there is already an assumption of trust among the nodes. Moreover, the alternatives like OSPF is even more complicated.

(?)


## Cloud CNI Plugins



# Service


# Getting traffic into the cluster

## Ingress

## Cloud Balancer

## metalLB


# In-cluster DNS


# Conclusion


# Other Reference

https://thenewstack.io/hackers-guide-kubernetes-networking/
Multiplexing and Hardware switching (SR-IOV, Single Root IO Virtualization) are two other methods

https://medium.com/flant-com/calico-for-kubernetes-networking-792b41e19d69
Sales piece for Calico

https://arthurchiao.art/blog/conntrack-design-and-implementation/
Connection Tracking (conntrack): Design and Implementation Inside Linux Kernel
A very detailed guide


[^1]: Node to node communication is a given assumption of the cluster itself. Pod to node communication is not specified because of the relation between Node and Pod: Node supervise the Pod and so the Pods are visible to the Node, but not vice versa - in normal situation they should be oblivious.
