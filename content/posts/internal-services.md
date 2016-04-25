+++
Categories = ["Development", "Microservices"]
Description = "Teams need the ability to access internal services easily"
Tags = ["Development", "microservices"]
date = "2016-04-24T11:30:14-07:00"
menu = "main"
title = "Accessing Internal Services"

+++

_TLDR: Teams need to have the ability to deploy and access internal services easily._

In the days before microservices, teams lived in the "Admin Section" of an app. It was the dumping ground for all functionality used the team that they didn't want users to see.

These days, that admin section should be a small constellation of microservices, focused entirely of functionality the developers and admins use to configure and  monitor the cluster. Additionally, it's not uncommon for apps to expose separate endpoints that expose stats or a simple API.

Accessing those internal services becomes critical. Teams might be tempted to resort to HTTP only, password protected APIs exposed on their public IPs. This is a security nightmare and should be discounted as soon as possible. Those internal APIs are never given the same security considerations as the public.

Mapping internal services to external, public ports also becomes an administrative headache. Every service needs a new port and keeping tracking of that port mapping and who is using which one (and thusly which ones need to be active) spirals out of control quickly.

### Providing Access

#### VPN

One of the easiest ways to solve this problem is to provide a VPN connection that developers can use to obtain access the internal cluster network, so that they may access the internal services.

If you're on a office network, perhaps even that office network is setup to allow routed traffic to your production network. If you already have this, great. But I actually don't recommend setting this up. It makes it much harder to track who has access to the internal services and it's possible you don't want everyone on that network to access them. Additionally, when developers aren't on that office network, they need another solution to access those resources anyway. For that reason, start and end with a good client VPN solution. Treat your office network like it's the wifi at a coffee shop.

There are many VPN solutions, scaling up in depending on your needs. Everything from [using a recent ssh's native interface tunnel](https://help.ubuntu.com/community/SSH_VPN) to [OpenVPN](https://openvpn.net) and up to commercial VPN appliances.

One solution that I stumbled upon and have been using more and more is [ZeroTier](https://www.zerotier.com). It's a peer-to-peer VPN which makes it more flexible (for instance to allow access to services provided by other clients) and the configuration is dead simple. They recently bumped up their free accounts to allow 100 devices and for $29.99 you can have unlimited devices. The client is [open source](https://github.com/zerotier/ZeroTierOne/) so what you're paying for is the convenience of using their cloud controller. But with the free accounts now allowing 100 devices, it's easy to see that everyone but large companies could use the service free!

ZeroTier also provides a public API to allow querying which devices are online and their connection information, making it easy to incorporate DNS and auditing systems. The client also has a local API that can be used control it. This makes it also easy to debug and script against, depending on needs.

Basically, ZeroTier is the ultimate power user VPN.

#### Tunneling

Another mechanism that can be used is programmatic tunneling. The best example of this is kubernetes [`kubectl port-forward`](http://kubernetes.io/docs/user-guide/kubectl/kubectl_port-forward/). This mechanism creates a tunnel and binds a local port to access a service running within a kubernetes Pod.

When a user needs to access a service, they simple run the right incantation of `kubectl port-forward` and then access the bound port. The traffic is tunnel over the kubernetes api service running within the cluster.

There is a big limitation with `kubectl port-forward` though. It can't access services, only pods. This means that the name of the thing to bind to is unstable (because pod names are usually transient, based upon the name of the replicated controller). So to use `kubectl port-forward` with any regularity, users have to script translating from a replicated controller or service to a pod, then invoke `kubectl port-forward`. I hope that in the future, this functionality is built directly in, it would vastly improve the usefulness.

### Directory

The number of revelation behind deploying microservices is that it's easy to lose track of them. With dynamic scheduling and replicas, what is running where becomes a task that a computer should be tracking, not a human.

This directory of services becomes important when developers want to use these internal services. Hunting around for random IP address / port combos is a great way to loose a day and then accidentally use the wrong service all together.

This directory concept directly fits in with my [previous post about bridging the gap between local and remote services](/posts/microservice-dev/). In this case though what is needed is basically an internal portal, listing the services and provide links to the internal HTTP ones. This concept is already done by larger engineering organizations but rarely makes it way down. Every team doing microservices, whether it's a one person operation or Google itself, needs this functionality.

### Case Study: Kubernetes + ZeroTier {#case-study}

I want to detail the setup that I currently use which is working fantastically, not only for developer access, but also to bridge services between clusters.

_The names of the services/clusters have been changed to protect the innocent_

#### The players

* Cluster W, running:
  * Reader
  * Purge
* Cluster M, running:
  * Members
* Developer machine Z

Both clusters are running kubernetes and there are associated replicated controllers and services for each of the above apps.

##### Installation

1. We need a ZeroTier network. Login and create one, giving it whatever name.
1. Install the ZeroTier clients on the master instances of W and M, but not the minions.
1. Add the master instances ZeroTier clients to your previously created network.
1. Configure the master instances to run `kube-proxy`. This means that the masters are now aware of all services in the cluster and can route traffic to the minions based on those services.
1. Install ZeroTier on Z and add it to the ZeroTier network.
1. At this point, you should have connectivity from Z to both W and M. The ZeroTier UI will show you the IP address that they've been assigned and you should be able to ping between them.
  * If you can't then it's likely that there are firewalls blocking traffic getting to W and M.
  * Alter the firewall (for instance, in AWS, change the security group) to allow UDP port 9993.
  * You should now have connectivity.
1. Z can now access any service provided by clusters W and M that use [NodePort]( http://kubernetes.io/docs/user-guide/services/#type-nodeport).

Because ZeroTier provides a Point-to-Point VPN, the Readers service now has access to Members on M. But because the traffic has to flow through the master, you need to add an IP route so that traffic for the ip subnet used by ZeroTier is sent to the master. On AWS, this is done by adding a route to your VPC.

##### Directory

To do basic dynamic configuration, the ZeroTier API is queried for devices and their assigned name is added to a special `int.blah.com` domain. That allows for device to easily access the other nodes by name rather than auto-assigned IP.

For the apps running inside the Kubernetes cluster, those are assigned ports and tracked by Kubernetes itself. There is a simple script that can resolve the node port assigned to a service from it's name. Because these ports are stable, they can be put into configuration files safely.

##### Day to day

From Z, it's possible to access the Members service simply by doing `curl w.int.blah.com:$(resolve m members)`, where `resolve` is a simple script that translates the cluster name and service into a port number.

Reader can use the same addressing mechanism to access Members as well.

### Summary

Internal services are backbone of a microservices architecture. They provide the most important functionality and accessing them is critical. Provide secure and consistent access to removes stumbling blocks, keeping everything moving.
