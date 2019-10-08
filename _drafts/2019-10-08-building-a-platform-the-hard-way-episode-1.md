---
layout: post
title: Building a platform the hard way: episode 1
---

Hi everybody welcome back to the `Building a platform the hard way` series. In this episode we'll be taking on the discovery component.

The sole purpose of a discovery component is to cope with the dynamic nature of modern infrastructure.
A service unique coordinates are described by its IP address and its listening port. Indeed, given an IP, only 1 service can listen on a given port.
Now services move places for various reasons but the most common being the inability to upgrade, in place the underlying hardware or runtime of the service, or even the service itself.
That's hoy the oldest discovery mechanism was born: DNS.
DNS had been around way before containers and the cloud. It allow a service consummer to be abstracted from the service location via a domain name and a port. 
That's exactly what happens when you visit a web page or write an email to a friend: one of the first thing to happen is to `resolve` the service's physical address.

Discovery is the same mechanism but at a lower scale, and in a closed environment: the whole internet does not need to find our services, only our peers. 




They are dynamic because of 2 main reasons:

- a service can be located on 
