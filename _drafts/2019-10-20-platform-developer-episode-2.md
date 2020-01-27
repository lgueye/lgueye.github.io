---
layout: post
title: Platform developer: episode 2
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

Infrastructure provisioning
===

Self-service infrastructure usually implies a cloud provider somewhere, be it public or private. 
Operating the cloud provider requires credentials. In the case of [Digital Ocean]() it's an API token.
Also, any provider somehow needs to configure the `root account` of the instance being created. It's usually the `root` ssh key pairs which will be uploaded in the `/root/.ssh` folder of the instance.
But these are credentials. You can't hardcode them nor include them in your terraform scripts since they're pushed in your SCM. You need to provide them in the command line and reuse them in the terraform scripts.

Terraform drives [Digital Ocean]() API.
You must install it.
```
$ terraform -version
Terraform v0.12.19

```

Your terraform command line could like this:

```
$ terraform plan -var "do_token=${DO_API_TOKEN}" -var "pub_key=${HOME}/.ssh/id_rsa.pub" -var "pvt_key=${HOME}/.ssh/id_rsa" ...
```

And the corresponding terraform script could look like this:

```
# provider variables
variable "do_token" {}
variable "pub_key" {}
variable "pvt_key" {}
variable "ssh_fingerprint" {}

provider "digitalocean" {
  token = var.do_token
}

```

In the above command line you can see that the credentials belong to the operator which runs terraform.
This is a problem because once the instance is created, only the operator will access it.
Meet your 1st collaboration issue. It needs to be addressed as soon as possible because any operator with power of the organization needs to be able to provision instances.

There are 3 common strategies which fix this collaboration issue:

- terraform enterprise
- a central machine (usually the Continuous integration machine) setup with shared ssh key (a home made version of the previous strategy which comes with its own challenges)
- maintain a list of ssh keys of users with power and add them to the instances (works for small teams but not really an option to scale)

Pick the strategy which best fits your organization and start developing provisioning scripts with terraform.

Coming back to our initial goal, we need to provision 8 compute instances. Providers have greatly simplified provisioning management.
One can specify physical specs (compute capacity, memory, storage), operating system image (ubuntu 18.04 for instance), datacenter location (london, frankfurt, etc), host name, root account, networking specs and meta data.
Below, an example of such declaration in terraform:

```
resource "digitalocean_droplet" "discovery_server_01_droplet" {
  image = var.droplet_image
  name = "${var.discovery_server_role}-01"
  region = var.primary_datacenter_name
  size = var.droplet_size
  private_networking = true
  ssh_keys = [
    "${var.ssh_fingerprint}"]
  tags = [
    "${var.target_env}",
    "${var.discovery_server_role}"]
}
```

By now we have described our infrastructure and terraform can execute that description.
The provider will assign IPs to each instance which will be collected (among other characteristics) in the provider http response.
These information will be codified in the terraform state file (`.tfstate`). This file captures the state of the infrastructure at the time terraform has executed.
It's also thanks to that state that terraform is idempotent.

Well you just got operational instances but frankly they're not so useful because the're just linux naked boxes.
You need to configure them and deploy useful applications. That's a job for the next tool in the chain: Ansible.


Application provisioning and deployments
===

In the previous chapter we've created instances and the result of the execution is stored in a `.tfstate` file. 
Ansible input is an inventory which describes a set of hosts organized in groups.
Each execution can match several groups which include the hosts under those groups. Matching supports intersections, union and wildcard. 
Unfortunately, Ansible does not understand `tfstate`. Nicolas Bering was kind enough to write 2 tools which address this issue
- a [terraform ansible provider](https://github.com/nbering/terraform-provider-ansible). The goal of the plugin is to extend terraform script by describing ansible hosts and ansible groups that will belong to the inventory
- a [terraform inventory](https://github.com/nbering/terraform-inventory) which dynamically generates an inventory from a `.tfstate` file.

The below snipet declares a host, the groups it belongs to and the vars associated to it
```
resource "ansible_host" "discovery_server_01_droplet" {
  inventory_hostname = digitalocean_droplet.discovery_server_01_droplet.name
  groups = [
    "${var.target_env}",
    "${var.discovery_server_role}",
    "${var.primary_datacenter_role}"]
  vars = {
    ansible_host = "${digitalocean_droplet.discovery_server_01_droplet.ipv4_address}"
    ansible_python_interpreter = "${var.ansible_python_interpreter}"
    datacenter_name = var.primary_datacenter_name
    datacenter_role = "${var.primary_datacenter_role}"
  }
}
```

This is how I use the inventory:
```
export ANSIBLE_TF_DIR=../terraform/ && ansible-playbook --vault-id ~/.ansible_vault_pass.txt -i /etc/ansible/terraform.py shared.yml -e "target_env=staging rev=`git log -1 --pretty='%h'`"
```

A lot happens in that command line:
- the export statement `export ANSIBLE_TF_DIR=../terraform/` is important for the inventory `-i /etc/ansible/terraform.py`. The inventory script downloaded from Nicolas Bering github space has been installed at `/etc/ansible/terraform.py`. It requires to know the location of the `.tfstate` directory.
- sensitive data are encrypted in the script and you need the password stored in `~/.ansible_vault_pass.txt` to decrypt them
- scripts need to know which revision we're cloning
- scripts need to know which env we're targeting. Each host in the inventory belongs to a `ènv` group making it easy to target all `staging` hosts for instance ignoring all `prod` (which can be helpful)
- shared.yml is the main script which contains the provisioning logic.

In order to write a proper Consul provisioning one must understand how it works. At the core:
- a servers cluster which maintain the cluster state in a key value store
- clients installed as side cars, that also belong to the cluster but don't manage the cluster state

You must write at least 2 provisioning scripts
- one to bootstrap the servers cluster.
- one to install a client on an arbitrary node. The consul client joins the servers and becomes a proxy to the servers.

In our example the following node are all consul client nodes:
- consumer node
- producer nodes
- test node

Ansible will install a consul client on those nodes because they were declared in the tfstate as belonging to a `discovery_client_role` group
```
- hosts: "&{{ target_env }}:&discovery-client"
  remote_user: "root"
  become: 'yes'
  roles:
  - { role: 'discovery/deploy_consul' }

# Upgrade consul on all clients
- hosts: "{{target_env}}:&discovery-client"
  remote_user: "root"
  become: 'yes'
  tasks:
  - block:
    - include_role: {name: 'oefenweb.dnsmasq'}
    - name: "[systemd-resolved vs dnsmasq conflict]: update local name server"
      lineinfile:
        path: /etc/resolv.conf
        regexp: '^nameserver 127.0.0.53'
        line: 'nameserver 127.0.0.1'
    - name: "[systemd-resolved vs dnsmasq conflict]: stop systemd-resolved.service"
      service: name="systemd-resolved" state='stopped'
    - name: "[systemd-resolved vs dnsmasq conflict]: disable systemd-resolved.service"
      service: name="systemd-resolved" enabled='false'
    - name: "[systemd-resolved vs dnsmasq conflict]: restart dnsmasq.service"
      service: name="dnsmasq" state='restarted'
    rescue:
    - fail:
        msg: "Failed to upgrade consul client"

- hosts: "&{{ target_env }}:&discovery-client"
  remote_user: "root"
  become: 'yes'
  roles:
  - { role: 'discovery/upgrade_consul', node_role: 'client'}

```

The server nodes were declared as belonging to a `discovery_server_role` group so they'll benefit from the server provisioning.
```
- hosts: 'localhost'
  connection: 'local'
  gather_facts: yes
  roles:
  - { role: 'discovery/download_consul' }

- hosts: "&{{ target_env }}:&discovery-server"
  remote_user: "root"
  become: 'yes'
  roles:
  - { role: 'discovery/deploy_consul' }
  - { role: 'discovery/upgrade_consul', node_role: 'server'}

```


A good implementation is always driven by a good use case.
The following use case seems quite relevant to demonstrate Consul Service Discovery capabilities/

- the server
  - 3 consul server nodes
  - responsible for maintaining consistent state in the whole cluster. The hole cluster is the set of all server nodes as well as all client nodes (agents)
- the clients
  - Service A: distributed on 2 nodes
  - Service B: distributed on 2 nodes, depends on service A

There are 3 main events in the regular lifecycle of this discovery mechanism:

- server cluster formation: happens at the very beginning, when server nodes try to find each other.
- client registration: when the client node starts, it makes the whole cluster aware that it is available to serve requests. it registers itself under a unique service name.
- client de-registration: when the client stops consul removes it from the service nodes. The service is still available: requests will be routed to other alive nodes of the service. A service will be considered down when all members are unreachable

The beauty of consul is that, as a developer, you need to worry about none of this mechanic. It is all taken care of. You need to describe a behavior, not to implement it.


