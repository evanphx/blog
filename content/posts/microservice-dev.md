+++
Categories = ["Development", "Microservices"]
Description = ""
Tags = ["Development", "Microservices"]
date = "2016-04-17T20:25:13-07:00"
menu = "main"
title = "Microservice Development - Who Runs What Where"

+++

_TLDR: The industry needs a common set of practices for how to develop microservices. This post discusses the required features those practices provide._

"Microservices!" she shouted, exclaiming what a brave, new world we were
living in.

"No more monoliths! Code bases so small you can fit them in the palm of your hand!". The dream was surely alive.

But as quickly as the exhuberance of a new development paradym set in, the
trouble began.

"Now instead of running one app to develop a feature, I need to have access to 5 different, coordinating services!"

Everyone that is doing microservices has this question. How this question is answered is as varied as there are teams. And so I posed the question on twitter:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">I’d love to hear how people developing services manage their dev environment for those services. Run everything locally? Nothing locally?</p>&mdash; Evan Phoenix (@evanphx) <a href="https://twitter.com/evanphx/status/721499732665720832">April 17, 2016</a></blockquote> <script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

The responses started to roll in:

<blockquote class="twitter-tweet" data-conversation="none" data-lang="en"><p lang="en" dir="ltr"><a href="https://twitter.com/evanphx">@evanphx</a> we usually run the service you&#39;re coding on locally, and run service dependencies in a shared sandbox environment.</p>&mdash; Evan Owen (@kainosnoema) <a href="https://twitter.com/kainosnoema/status/721501519523098624">April 17, 2016</a></blockquote>

<blockquote class="twitter-tweet" data-conversation="none" data-lang="en"><p lang="en" dir="ltr"><a href="https://twitter.com/evanphx">@evanphx</a> Everything locally in separate VMs (if there are dependencies) created from provisioning scripts and mock data.</p>&mdash; Patrick Lenz (@patricklenz) <a href="https://twitter.com/patricklenz/status/721574452677472256">April 17, 2016</a></blockquote>

<blockquote class="twitter-tweet" data-conversation="none" data-lang="en"><p lang="en" dir="ltr"><a href="https://twitter.com/evanphx">@evanphx</a> we run everything locally.</p>&mdash; Mike Chlipala (@mikec) <a href="https://twitter.com/mikec/status/721504251407519744">April 17, 2016</a></blockquote>

<blockquote class="twitter-tweet" data-conversation="none" data-lang="en"><p lang="en" dir="ltr"><a href="https://twitter.com/evanphx">@evanphx</a> Run everything locally. Don’t test locally though. For testing (incl. performance) hit test services which mirror prod, not staging</p>&mdash; Jeremy Tregunna (@jtregunna) <a href="https://twitter.com/jtregunna/status/721500756294115328">April 17, 2016</a></blockquote>

<blockquote class="twitter-tweet" data-conversation="none" data-lang="en"><p lang="en" dir="ltr"><a href="https://twitter.com/evanphx">@evanphx</a> we run only the service under development locally. Dependencies are run in a sandbox on our PaaS (Cloud Foundry in our case)</p>&mdash; Bruce E. Thelen (@brucethelen) <a href="https://twitter.com/brucethelen/status/721668455074672640">April 17, 2016</a></blockquote>

There seem to be 2 schools represented:

* Run everything locally
  1. in VMs
  1. in containers
  1. as regular programs
* Run only one service locally, the rest in a remote sandbox

I'd imagine that the second case is a reaction to the complicated nature of managing the first. If 5 different services are required to do development, a developer can easily lose a week just trying to get everything setup. VMs and containers can certainly help, and help a lot, but they still put heavy resource constraints upon a single machine.

What microservice development needs as a mainstream strategy that can unify these 2 approaches so that developers can easily flow between them based on ease of use. Namely, these are the necessary features:

* Ability for multiple services running locally to find each other
* Ability for local services to use remote ones
* Ability for local services to give out a url/address for themselves that remote services can use

Some features that I think would make such a tool/strategy even better:

* A monitor that can report on traffic flowing between services
  * Could even include the full data stream, ala ngrok's HTTP monitor support
* Local log aggregation
* Ability to easily rebuild and restart one service without disrupting other local services

Let's breakdown these features:

### Multiple Local Services

The easiest way do this is static port mappings. Service A always runs on 20001, B on 20002, etc. You can get a long ways doing this, but obviously it breaks down in B is now being run remotely instead of locally. In that case, port 20002 needs to run some kind of proxy that can route traffic to B.

One upside of static port mappings is that it simplifies internal service discovery. Rather than needing to be coded with a specific dynamic discovery strategy (DNS, Consul, etcd, etc), a simple config file or even static configuration can be used.

A place where static port mappings breaks down badly is when 2 versions of a service need be used. Reverting locally changes to a service to fix an unrelated bug is a huge productivity killer. For that reason, some level of dynamic configuration is preferable. It doesn't have to be runtime dynamic, for instance environment variable inject counts as dynamic in this context.

### Dynamic Service Configuration

There are 2 elements to dynamic service configuration that we should discuss.

First is what details about a service are advertised and how the systems knows that info. For all local services, that could be a simple config file or a trivial local database where services register themselves. When you add remote services into the mix as well, something like Consul, etcd, or even Kubernetes can be queried. The key here is that a program running on the local machine can answer based on local values as well as remote ones.

The second part is how service information is consumed. Simplest is environment variable injection. To find service B, simply read `SERVICE_B_ADDRESS`. This is a tried and true solution, and gives services a high degree of flexibility with regard to how they integrate that information into their local config. But environment variables have a huge downside: they can only be set at program start. That means that all services must be known and assigned some static value before any program starts up.

Another common technique is to use DNS to provide service information. Consul is a great example of that, it provides SRV records for services, so that given a name it's possible to get the host and port to connect to. To use DNS service discovery though, either the DNS server that can answer with the service records needs to be in `/etc/resolv.conf` or the DNS server needs to queried it directly. The later option is basically bootstrapping DNS based discovery via an environment variable: `SERVICE_DNS_ADDRESS` is injected as an environment variable, then queried to provide any further records.

_A side note on DNS:_ Because we want to allow for dynamic port allocation for added flexibility, a service will likely need to query SRV records, not just an A record, from the DNS server. Because of that `gethostbyname` can't be used, even if the DNS server is configured properly via `/etc/resolv.conf`. For that reason, it's almost always cleaner to inject the DNS server address via an environment variable rather than fiddle with `/etc/resolv.conf`.

### Connectivity and Callback URLs

Assuming we've wired up the above functionality, local services can now find other services, regardless of if they're local or remote. Now the rubber meets the road, services need to connect and send requests to each other. If a service is remote, either there is a VPN pre-configured to allow local service to route to it, or another tool is going to be running locally and providing some kind of proxy. Getting into all of that is a post for another day, so we'll assume that is already setup. What I do want to go over briefly is the requirement that local services can be connected back to. Some will see this requirement as unnecessary, but if you're building asynchronous, HTTP-based services, it's- extremely useful because it means having to run and configure fewer services locally.

Providing this functionality can be accomplished by integrating functionality similar to [ngrok](https://ngrok.com). Services will need to have a way to find out their external address so they can give it out. That can happen by injecting the information via the same service configuration we used above. So in that way, the callback proxy is just another service that assists in bridging the local/remote gap.

## Additional Features

### Traffic Monitoring

In development, answering the question "what exactly did I just send or receive?" is extremely common. So common that it's common for people to just add debugging code do print out these values before being processed. Having a tool that is capturing that data for analysis would be super valuable.

### Local Log Aggregation

If a user is running 3 services locally, seeing the logs for each of them interlaced makes it much easier to understand what is happening. What I think is also important is that this log aggregation isn't the ONLY way to see the logs. It's also very useful to be able to just see the logs for one service without the noise of the others.

### Service Restarts

Programs are going to get restarted really often in development as they change. Relying on frameworks to reload code (such as in Rails) is not a solution to the problem, and so providing the user the ability to stop, rebuild, and start a service without disrupting all the other features here is critical.

This means:

* Assigned ports must not change between restarts
* Restarting all services to restart one is unacceptable

Development is all about introduce new, iterative versions of services and thusly easy restarting is a critical function.


## Summary

Microservice development is still in it's infancy. Teams are slowly figuring out solutions to their specific problems and largely hacking something together. If microservices are really going to be the future (and some would say the present), then these issues have to be solved.
