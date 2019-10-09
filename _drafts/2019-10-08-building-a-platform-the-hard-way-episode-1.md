---
layout: post
title: Building a platform the hard way: episode 1
---

Hello everybody welcome back to the `Building a platform the hard way` series. In this episode we'll be taking on the discovery component.

In an IT system, services can communicate together because they can find each other. There are situations though when consumers will not be able to locate a service:

- the network connection is broken: not a discovery issue per say.
- the service coordinates changed for various reasons. For instance the inability to upgrade the service in-place (service will be hosted on a machine which holds a different IP address which is ALWAYS true with containers)

To lower the risk of downtime, services are scaled: the service is no longer 1 instance but potentially N. The nice side effect of the scaling is the ability to apply new agile release strategies such as `blue green` or `canary`.

The benefit introduced for the release strategies and the scaling has become quite a complication for the consumers.
Consumers used to have 1 coordinate (IP / domain) associated to the service. They now need to maintain N coordinates. Keeping in sync consumers will prove close to impossible as services increase their release frequency.

Maybe we can do better ? Maybe we've got the paradigm wrong from the start: clients should not have to know about hardcoded coordinates. 

The strategy I just described is called [Client Discovery](https://microservices.io/patterns/client-side-discovery.html): clients gets a list of coordinates at startup and provide a way to update the list.

Taking a closer look at the problem, the issue looks a lot like what DNS wanted to address back in the eighties. It's a good start but it's not quite the same because DNS can provide you with the same coordinate given a domain name. 
What we need is the same feature but with a `random` answer on the set of elligible instances (`ready` and `live`).
This way the client maintains a service identifier (which can be a domain name in the DNS semantic but not) rather than a list of instances and instances are free to change without impacting clients. 

This strategy is called [Server Discovery](https://microservices.io/patterns/server-side-discovery.html).

Enough with the problem, I think you have a good grasp on what's at stake. Also 




Before modern service discovery solutions, people used to 

The sole purpose of a discovery component is to cope with the dynamic nature of modern infrastructure.
Now, services move places for various reasons but the most common being the inability to upgrade in-place the underlying hardware or runtime of the service, or even the service itself.

A service unique coordinates are described by its IP address and its listening port. Indeed, given an IP, only 1 service can listen on a given port.


That's why the oldest discovery mechanism was born: DNS.
DNS has been around way before containers and the cloud. It allows a service consummer to be abstracted from the service location via a domain name and a port.
That's exactly what happens when you visit a web page or write an email to a friend: one of the first thing to happen is `IP resolution` to be able to reach the service.

Discovery is the same mechanism but at a lower scale, and in a closed environment: the whole internet does not need to find our services, only our peers. 




They are dynamic because of 2 main reasons:

- a service can be located on 
