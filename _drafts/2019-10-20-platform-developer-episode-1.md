---
layout: post
title: Platform developer: episode 1
---

Hello everybody welcome back to the `Platform developer` series. In this episode we'll be taking on the discovery component.

The problem
====

In an IT system, services can communicate together because they can find each other. There are situations though when consumers will not be able to locate a service:

- the network connection is broken (not a discovery issue per say).
- the service coordinates changed for various reasons (service relocated on another machine which is ALWAYS true and very FREQUENT in containerized envs)

To lower the risk of downtime, services are scaled: the service is no longer 1 instance but potentially N. The nice side effect of the scaling is the ability to apply new agile release strategies such as `blue green` or `canary`.

The benefit introduced for the release strategies and the scaling has become quite a complication for the consumers.
Consumers used to have 1 coordinate (IP / domain) associated to a service. They now need to maintain N coordinates. Keeping in sync consumers will prove close to impossible as services increase their release frequency.

Maybe we can do better ? Maybe we've got the paradigm wrong from the start: clients should not have to know about hardcoded coordinates. 

The strategy I just described is called [Client Discovery](https://microservices.io/patterns/client-side-discovery.html): clients gets a list of coordinates at startup and provide a way to update the list.

Taking a closer look at the problem, the issue looks a lot like what DNS wanted to address, which is already excellent but not enough with the dynamic nature of modern infrastructures. 
In addition to 
What we need is the same feature but with a `resolution strategy` (random, least connections, sticky) on the set of elligible instances (`ready` and `live`).
This way the client maintains a service identifier (which can be a domain name in the DNS semantic but not) rather than a list of instances and instances are free to change without impacting clients. 

This strategy is called [Server Discovery](https://microservices.io/patterns/server-side-discovery.html).

Enough with the problem, I think you have a good grasp on what's at stake.

Possible solutions
===

No need to hide it anymore, I'm definitely in favor of the Server Discovery strategy because it is decoupled from the clients.
As I mentioned earlier, the discovery mechanism works with a naming convention between clients and servers.

Discovery solutions mainly emerged with containerized envs but some are not limited to. For instance [Consul](https://www.consul.io/) can be used in both containerized and classic environments.

The [Hashicorp Consul](https://www.consul.io) team has done a much better job at comparing service discovery solutions. I strongly advice you to visit the [Consul vs other software](https://www.consul.io/intro/vs/index.html) page. 


My advice
===

I encourage teams to stick with their platform default if applicable. Containerized platforms come with a default, better use it whenever possible, after all it's one less component to maintain (the best code is the one you don't maintain)

As for non containerized envs I always go for [Hashicorp Consul](https://www.consul.io) for the reasons below:
- I like the reliability, performance and rich feature set (DNS, side-car, strong consistency).
- The side-car pattern is smart for distribution
- Such a complex software in a single binary, a single command line and sane defaults ? I mean, they set the bar quite high and you can't deny that they DID put themselves in the shoes of the user to ease adoption. But this is not new for Hashicorp, all their products follow the same philosophy
- Even the opensource UI is high-quality, this extra effort needs our acknowledgement


Example implementation of Consul
===

If your were to evaluate [Hashicorp Consul](https://www.consul.io) I would suggest to follow the steps I mentioned in [episode 0](https://lgueye.github.io).

The definition of done could be to have a consumer which is able to find a producer service through consul.
We could go further by shutting down 1 instance of the consumer should still be able to find the producer service
All in all, to achieve this basic evaluation, we would need 8 compute instances:
- 3 consul server nodes
- 2 consumer node
- 2 producer nodes
- 1 test node (runs scenarii at startup)

Provision
===

Since the comppute instance run in [Digital Ocean](), the api key and secret are needed to execute [Hashicorp Terraform](), our provisioning tool.

The goal is to:
- describe the desired infrastructure in various `.tf` files
- successfuly run `terraform plan`
- successfuly run `terraform apply`
- then later run `terraform destroy`



[Hashicorp Terraform]() will collect the output of those instances creation and write it in a `.tfstate` file.

The format of that file can't be used as is by the next tool in the chain: [Ansible]() (the orchestration tool)
Nicolas Berring created a terraform plugin that extends the input `.tf` files. The extension allows to 
- collect generated IPs
- name a host
- assign the host to groups


to the newly cre `.tfstate` relevant sections and generates an Ansible inventory which is the scope (collection of nodes organized in groups) against which all orchestration operation apply


Orchestrate deployments
===

The plugin also creates a `dynamic inventory` from the exectution output (`.tfstate` file)
That inventory becomes the input of an



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


