---
layout: post
title: Building a platform the hard way: episode 1
---

Hello everybody welcome back to the `Building a platform the hard way` series. In this episode we'll be describing the steps required to implement a platform component.
After implementing numerous infrastructure components, the experience shows that, no matter the component, we can always breakdown its implementation in 5 main steps:

- provision
- secure
- orchestrate deployments
- integrate consumers
- test

Let's take a closer look at each step responsibility.

Provision
====

To understand what needs to be done in the scope of provisioning you must put yourself in the following situation:
- you belong to a team in an organization
- you need some work to be done
- you start some analysis
- you are ready to experiment
- you need a set of resources for that (multiple servers, guaranteed connectivity between them, security, etc)

So basically provisioning answers those needs: how do we, as a team, get to have a set of infrastructure ressources in the most agile possible way while implementing to the company compliance constraints.
Agile is key: from one company to another the setup of a new set of resources along with their configuration can take up to several weeks against minutes in most performant ones.

These situations best describe the above point:
- your organization is ticket based: you file several tickets to your ITSM (IT service Management) which will create virtual machines for your team, secure the machines, integrate them in the existing infrastructure
- your organization is self-service based: you describe your infrastructure requirements in a git branch, commit it and make a pull request, infrastructure team and security team review it, possibly add security aspects

Given your IT is self service, we could imagine that an operator would apply your requirements against your infrastructure provider(s) and get the job done.
Within minutes, your infrastructure would be ready to serve its purpose.
The operator can be a CI agent (jenkins/travis/whatever) or a user with he right level of trust. We'll see below why it matters

Terraform is the component I'm using to provision my instances in Digital Ocean.
The idea of terraform is quite simple:
- plan your intention. Given your description it'll compute the state of your infrastructure. Anything that already exists will be left untouched, anything that is not there yet will be added and anything present in the current infra that is absent from your description is going to be removed. Teraform is idempotent: describe your intention, it'll infer the required work. So neat.
- apply. Execute the execution plan inferred in the step above
- destroy. You're done with your lab session, well tear down all resources, your wallet will thank you.

Terraform will codify the state of its execution in a file. You will want to save that file either in your SCM (Source control management) if you trust it, or in a remote store that you trust.
It's important if you wish to infer the correct state on the next execution. It's also a convenient way to collaborate: remember that any operator can update the infrastructure.

Secure
====

In practice, a provisioning tool runs on behalf of an operator (be it human or not) machine. 
You'll need to provide the initial set of ssh keys. Those will automatically be associated to the root account. You see what's at stake here: which keys are going to be part of the closed circle of root authorized keys ?
Those keys need to belong to `trusted operators`. And you need an easy way to manage them: onboard new members, revoke existing ones.
There you go: meet your first secret management issue.
The way you solve it largelly depends on your organization security requirements but security is the one field you don't want to take lightly, you would be smart to leave it to seasoned experts.
Getting security right is not simple issue because on on hand it MUST not get in the way, on the other hand it must guarantee identity.

SSH keys are not the only type of security aspect that your organization will need to manage.  It's just the begining of a long list of identity management:
- ssh keys
- ssl certificates
- data store credentials
- ...

You'll want a solid automated workflow that will allow you to almost forget about it.

I'm not a security expert but it seems that Hashicorp Vault is able to manage all those secret, audit your access, expire them automatically and even provide one shot credentials.
I've never witnessed it but it looks like the de-facto industry standard.

Orchestrate deployments
===


The goal of a deployment is: given release N is currently RUNNING in production you want release N+1 to replace it as smoothly as possible.
You mihght baldly replace N by N+1 but this won't do your organization any good because you'll introduce downtime.
On the other hand upgrading with zero downtime is the GRAAL of Agile IT because the pre-requisites to that are very demanding.

At the core of it:
- upgrade DB or any other dependencies like queues, topics, etc: requires that the upgrade of the dependencies to be backward compatible because remenber version N is still running 
- for each node, node after node:
-- deploy version N+1: N+1 is started but is not receiving any traffic
-- drain version N
-- switch traffic to N+1
-- stop version N
-- check compliance (metrics, logs, security, etc) 

You definitely don't want to do that manually for each node required for your release.
You can codify these steps with suitable tools. My tool of choice is Ansible but you can choose any tool that makes you confident and comfortable
The alternative is to take advantage of your platform which usually implements various release strategies. But moving to such platforms comes at a high price.

Integrate consumers
===


Test
===

