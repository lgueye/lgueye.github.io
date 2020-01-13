---
layout: post
title: Building a platform the hard way: episode 1
---

Hello everybody welcome back to the `Building a platform the hard way` series. In this episode we'll be taking on the discovery component.

The problem
====

In an IT system, services can communicate together because they can find each other. There are situations though when consumers will not be able to locate a service:

- the network connection is broken: not a discovery issue per say.
- the service coordinates changed for various reasons. For instance the inability to upgrade the service in-place (service will be hosted on a machine which holds a different IP address which is ALWAYS true with containers)

To lower the risk of downtime, services are scaled: the service is no longer 1 instance but potentially N. The nice side effect of the scaling is the ability to apply new agile release strategies such as `blue green` or `canary`.

The benefit introduced for the release strategies and the scaling has become quite a complication for the consumers.
Consumers used to have 1 coordinate (IP / domain) associated to the service. They now need to maintain N coordinates. Keeping in sync consumers will prove close to impossible as services increase their release frequency.

Maybe we can do better ? Maybe we've got the paradigm wrong from the start: clients should not have to know about hardcoded coordinates. 

The strategy I just described is called [Client Discovery](https://microservices.io/patterns/client-side-discovery.html): clients gets a list of coordinates at startup and provide a way to update the list.

Taking a closer look at the problem, the issue looks a lot like what DNS wanted to address back in the eighties. It's a good start but it's not quite the same because DNS can provide you with the same coordinate given a domain name. 
What we need is the same feature but with a `resolution strategy` (random, least connections, sticky) on the set of elligible instances (`ready` and `live`).
This way the client maintains a service identifier (which can be a domain name in the DNS semantic but not) rather than a list of instances and instances are free to change without impacting clients. 

This strategy is called [Server Discovery](https://microservices.io/patterns/server-side-discovery.html).

Enough with the problem, I think you have a good grasp on what's at stake.

Possible solutions
===

No need to hide it anymore, I'm definitely in favor of the Server Discovery strategy because it is decoupled from the clients.
As I mentioned earlier, the discovery mechanism works with a naming convention between clients and servers.

Discovery solutions mainly emerged with containerized envs but some are not limited to. For instance [Consul](https://www.consul.io/) can be used in both containerized and classic environments.

Find below, a summary of the possible solutions

| Product  | Supported platforms | Pros| Cons |
| ------------ | --- |--- |--- |
| [Kube DNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) | Kubernetes | Available as a plugin | Requires K8s |
| [Overlay network](https://docs.docker.com/network/overlay/) | Docker Swarm | Available for free in Swarm | Requires Docker swarm |
| [Apache Zookeeper](https://zookeeper.apache.org/) | All | Available on all platforms | Hard to configure |
| [Eureka](https://github.com/spring-cloud/spring-cloud-netflix/tree/master/spring-cloud-netflix-eureka-server) | All | Available on all platforms | consumers need to include the eureka client library |
| [Consul](https://www.consul.io/) | All | Available on all platforms | Still looking for a cons :) |

My advice
===

I strongly encourage teams to stick with the platform default. Containerized platforms come with a default, better use it whenever possible, after all it's one less component to maintain

As for non containerized envs I always go for Consul.

Consider situation 1: 
- your team manages 20 backends without any service discovery (nginx is used to abstract their locations)
- you decide to introduce a service discovery to simplify your backends management
- you install Eureka server on 2 nodes (automated)
- youd need to change all consumers to include Eureka Client library and change dependencies configuration to match Eureka convention

Consider situation 2: 
- your team manages 20 backends without any service discovery (nginx is used to abstract their locations)
- you decide to introduce a service discovery to simplify your backends management
- you install Consul Server on 3 nodes (automated)
- you install Consul Agent on all consumers hosts (automated)
- youd change all consumers dependencies configuration to match Consul convention (dns).

Which one would you prefer ? 

I personally find Consul less intrusive and closer to the standard: as I said, DNS has been around for a while.
In addition, consul is consistent because it's based on [Raft Consensus Algorithm](https://raft.github.io/). Raft is out of the scope of this post but, in a nutshell, it provides strong guaranties on a cluster state: all cluster consumers see the same information which is always more reliable.

Example implementation with Consul
===

A good implementation is always driven by a good use case.
The following use case seems quite relevant to demonstrate Consul Service Discovery capabilities/

- the server
  - 3 consul server nodes
  - responsible for maintainig consistent state in the whole cluster. The hole cluster is the set of all server nodes as well as all client nodes (agents)
- the clients
  - Service A: distributed on 2 nodes
  - Service B: distributed on 2 nodes, depends on service A

There are 3 main events in the regular lifecycle of this discovery mechanism:

- server cluster formation: happens at the very begining, when server nodes try to find each other.
- client registration: when the client node starts, it makes the whole cluster aware that it is available to serve requests. it registers itself under a unique service name.
- client de-registration: when the client stops consul removes it from the service nodes. The service is still available: requests will be routed to other alive nodes of the service. A service will be considered down when all members are unreachable

The beauty of consul is that, as a developer, you need to worry about none of this mechanic. It is all taken care of. You need to discribe a behavior, not to implement it.


