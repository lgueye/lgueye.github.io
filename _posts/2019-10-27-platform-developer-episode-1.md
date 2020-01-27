---
layout: post
title: Platform developer: episode 1
---

Hello everybody welcome to the `Platform developer` series. In this series I'm willing to share my past experience on building quite a few platform components. 

I tried to follow the same development steps for all my components which I'll share with you in this series.
This episode will describe the rationale behind each step.

My experience shows that, no matter the component, I could always breakdown its initial implementation in 5 main steps:

- provision
- secure
- orchestrate deployments
- integrate consumers
- test

Let's take a closer look at each step.

Infrastructure provisioning
====

To understand what needs to be done in the scope of provisioning you must put yourself in the following situation:
- you belong to a team in an organization
- you need some work to be done
- you start some analysis
- you are ready to experiment and challenge your ideas
- you need a set of resources to start crafting (multiple servers, guaranteed connectivity between them, etc)

Provisioning basically answers those needs: how do we, as a team, get to have a set of infrastructure resources in the most agile possible way while implementing the company compliance constraints.
Agile is key: from one company to another the setup of a new set of resources along with their configuration can take up to several weeks against minutes in most performing ones.

These situations best describe the above point:
- your organization is ticket based: you file several tickets to your ITSM (IT service Management) which will create virtual machines for your team, secure the machines, integrate them in the existing infrastructure
- your organization is self-service based: you describe your infrastructure requirements in a git branch, commit it and make a pull request, infrastructure team and security team review it, possibly add security aspects. You make another pull request and the infrastructure is all gone.

Given your IT is self service, we could imagine that an operator would apply your requirements against your infrastructure provider(s) and get the job done.
Within minutes, your infrastructure would be ready to serve its purpose.
The operator can be a CI agent (jenkins/travis/whatever) or a user with he right level of trust. We'll see below why it matters

[Terraform](https://www.terraform.io) is the component I'm using to provision my instances in [Digital Ocean](https://cloud.digitalocean.com/login).
The idea of [Terraform](https://www.terraform.io) is quite simple:
- plan your intention. Given your description it'll compute the state of your infrastructure. Anything that already exists will be left untouched, anything that is not there yet will be added and anything present in the current infra that is absent from your description is going to be removed. [Terraform](https://www.terraform.io) is idempotent: describe your intention, it'll infer the required work. Neat.
- apply. Execute the execution plan inferred in the step above
- destroy. You're done with your lab session, well tear down all resources, your wallet will thank you.

Terraform codifies the state of its execution in a file. You will want to save that file either in your SCM (Source control management) if you trust it, or in a remote store that you trust.
It's important if you wish to infer the correct state on the next execution. It's also a convenient way to collaborate: remember that other operators also need to update the infrastructure.

To install [Terraform](https://www.terraform.io) you need to follow your platform (Windows/Linux/MacOS) instructions (available on [Terraform](https://www.terraform.io) website).
A proper installation should have a similar output
```
$ terraform -version
Terraform v0.12.19

```

Self-service infrastructure usually implies a cloud provider somewhere, be it public or private. 
Operating the cloud provider requires credentials. In the case of [Digital Ocean](https://cloud.digitalocean.com/login) it's an API token.
In addition, any provider somehow needs to configure the `root account` of the instance being created. It's usually the `root` ssh key pairs which will be uploaded in the `/root/.ssh/` folder of the instance (usually `authorized_keys` file).
But these are credentials. You can't hard-code them nor include them in your terraform scripts since they're pushed in your SCM. You need to provide them in the command line and reuse them in the terraform scripts.

For [Digital Ocean](https://cloud.digitalocean.com/login), your [Terraform](https://www.terraform.io) command line could look like this:
                  
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

In the above command line you can see that the credentials belong to the operator which runs [Terraform](https://www.terraform.io).
This is a problem because once the instance is created, only the operator will be able to access it.

Meet your 1st collaboration issue. This issue is serious enough for the team to quickly find a decent solution before moving to next steps.

There are 3 common strategies which fix this collaboration issue:

- [Terraform Enterprise](https://www.terraform.io/docs/enterprise/index.html)
- a central machine or set of machines (usually the Continuous integration machines) setup with shared ssh key. This is  a home made version of the previous strategy which comes with its own challenges. Those CI instance become the only ones being able to operate the cloud provider. They must be treated with extreme care and caution.
- maintain a list of ssh keys of users with power and add them to the each instance you create. Works for small teams but not really an option to scale

Pick the strategy which best fits your organization and start developing scripts with [Terraform](https://www.terraform.io).

When you've addressed the shared credentials issue, you need to create compute instances
Providers have greatly simplified provisioning automation.
One can specify physical specs (compute capacity, memory, storage), operating system image (ubuntu 18.04 for instance), datacenter location (london, frankfurt, etc), host name, root account, networking specs and meta data.
Below, an example of such declaration in [Terraform](https://www.terraform.io):

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

By now we have described an executable version of our infrastructure with [Terraform](https://www.terraform.io) as the execution agent.

Upon execution, [Terraform](https://www.terraform.io) will persist the state of your infrastructures (hosts, Ips, etc) in a file, the `.tfstate`.
It's thanks to that state that [Terraform](https://www.terraform.io) is idempotent: if the file has not changed, and your intent is the same, next time you run [Terraform](https://www.terraform.io) it simply do nothing.

Well, you just got operational instances but frankly they're not so useful because the're just linux naked boxes.
To leverage this work you need to configure them and deploy useful applications. That's a job for the next tool in the chain: [Ansible](https://docs.ansible.com/ansible/latest/index.html).
But first a word on security

Infrastructure security
====

Tackling the root account SSH key issue was the 1st security challenge but SSH keys are not the only security aspect that your organization will need to manage.  
It's just the beginning of a long list of identity management:
- ssl certificates
- datastore credentials
- ...

You'll want a solid automated workflow that will not get in your way.

I'm not a security expert but it seems that Hashicorp Vault is able to manage all those secret, audit your access, expire them automatically and plug to identity management systems such as LDAP. It becomes the enterprise Authorization Management system. 
I'm not a Vault expert even less security expert but it looks like it's becoming the de-facto industry standard.


Application provisioning and deployments
===

The goal of a deployment is, given release N is currently RUNNING in production, to upgrade to release N+1 without downtime.
You might baldly replace N by N+1 but this won't do your organization any good because you'll introduce downtime.
On the other hand upgrading with zero downtime is the GRAAL of Agile IT because the pre-requisites to that are very demanding.

At the core of it you would:
- upgrade DB or any other dependencies like queues, topics, etc: requires that the upgrade of the dependencies to be backward compatible because version N is still running 
- for each node, node after node:
  - deploy version N+1: N+1 is started but is not receiving any traffic
  - drain version N
  - switch traffic to N+1
  - stop version N
  - check compliance (metrics, logs, security, etc) 

You definitely don't want to do that manually for each component.
You can codify these steps with suitable tools. My tool of choice is [Ansible](https://docs.ansible.com/ansible/latest/index.html) but you can choose any tool that makes you confident and comfortable
The alternative is to take advantage of your platform (K8s) which usually implements various release strategies. But moving to such platforms comes at a high price.

In provisioning step we've created instances and the result of the execution is stored in a `.tfstate` file. 
[Ansible](https://docs.ansible.com/ansible/latest/index.html) input is an inventory which describes a set of hosts organized in groups.
Each execution can match several groups which include the hosts under those groups. Matching supports intersections, union and wildcards. 
Unfortunately, [Ansible](https://docs.ansible.com/ansible/latest/index.html) does not understand `tfstate`. Nicolas Bering was kind enough to write 2 tools which address this issue
- a [terraform ansible provider](https://github.com/nbering/terraform-provider-ansible). The goal of the plugin is to extend terraform script by describing [Ansible](https://docs.ansible.com/ansible/latest/index.html) hosts and [Ansible](https://docs.ansible.com/ansible/latest/index.html) groups that will belong to the inventory
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
- scripts need to provide an execution context (env, revision, etc). Each host in the inventory belongs to a `ènv` group making it easy to target all `staging` hosts for instance ignoring all `prod` (or other envs)
- shared.yml is the main script which contains the provisioning logic.

All [Ansible](https://docs.ansible.com/ansible/latest/index.html) commands follow the same structure, the only things that will vary are the `.yml` file and the env vars.


Consumers integration
===


Deploying your components is by no means the finality. You deployed them for a set of consumers.
I believe it's a good practice to put yourself in the shoes of your consumers to feel their experience.
A good way to do so is to create clients. Providing clients has obvious advantages but can prove quite challenging.

The good parts of client is obviously the reusability and consistency: anyone willing to interact with your component can include the client instead of each consumer rolling its own version of a client.

The biggest challenge of clients is suitability: when a client is not suitable for your consumer usage you need to evolve it and this is the start of potential issues:
- compatibility: you must design backward compatible clients. You would not want to satisfy one client and make others unhappy
- flexibility: the client must be flexible enough. For instance some consumers might be interested in a dry-run and should be able to configure it easily

The last bit of the integration is a realistic runnable example which can take the form of a test.
I'm not going to remind you the numerous benefits of automated tests. They usually take time to setup but the benefits vastly outweighs the investment in the long run.

Find below an example of a rabbitmq consumer which uses a java client:
```
/**
 * Queues configuration are shared by all consumers
 * Therefore we do not need to declare them in each consumer
 * This common setup applies to all of them
 * If the config is already present in rabbitmq it wont be created again
 *
 * @author louis.gueye@gmail.com
 */
@Configuration
@Slf4j
public class PlatformBrokerClientConfiguration {

	@Bean
	public MessageConverter messageConverter() {
		return new Jackson2JsonMessageConverter(Jackson2ObjectMapperBuilder.json() //
				.serializationInclusion(JsonInclude.Include.NON_NULL) // Don’t include null values
				.featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS) // use ISODate and not other cryptic formats
				.featuresToDisable(SerializationFeature.FAIL_ON_EMPTY_BEANS) // allow empty json to be produced (introduced with care alarm models
				// which can be as simple as `{}`
				.featuresToDisable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES) // do not fail on unknown properties because they are likely to
				.modules(new JavaTimeModule()) //
				.build());
	}

	@Bean
	public PlatformBrokerClient brokerClient(final AmqpTemplate amqpTemplate) {
		return new PlatformBrokerClient(amqpTemplate);
	}

	@Bean
	public QueuesConfig queuesConfig() {
		return new QueuesConfig();
	}

	@Bean
	public List<Declarable> directBindings(final AmqpAdmin amqpAdmin, final QueuesConfig queuesConfig) {
		log.info("Creating direct exchanges...");
		final List<Declarable> declarables = Lists.newArrayList();
		if (queuesConfig == null || CollectionUtils.isEmpty(queuesConfig.getExchanges()))
			return declarables;
		queuesConfig.getExchanges().forEach(exchange -> {
			Exchange ex = ExchangeBuilder.directExchange(exchange.getId()).durable(true).build();
			amqpAdmin.declareExchange(ex);
			declarables.add(ex);
			log.info("Successfully created exchange {} ({}).", ex.getName(), ex.getType());
			exchange.getRoutes().forEach(queue -> {
				Queue q = QueueBuilder.durable(queue.getId()).build();
				log.info("Successfully created queue {}.", queue.getId());
				amqpAdmin.declareQueue(q);
				declarables.add(q);
				Binding b = BindingBuilder.bind(q).to(ex).with(queue.getKey()).noargs();
				declarables.add(b);
				amqpAdmin.declareBinding(b);
				log.info("Successfully bound exchange {} to queue {} with routing key {}.", exchange.getId(), queue.getId(), queue.getKey());
			});
		});
		return declarables;
	}

	@Bean
	public TopicsConfig topicsConfig() {
		return new TopicsConfig();
	}

	@Bean
	public List<Declarable> fanoutBindings(final AmqpAdmin amqpAdmin, final TopicsConfig topicsConfig) {
		log.info("Creating fanout exchanges...");
		final List<Declarable> declarables = Lists.newArrayList();
		if (topicsConfig == null || CollectionUtils.isEmpty(topicsConfig.getExchanges()))
			return declarables;
		topicsConfig.getExchanges().forEach(exchange -> {
			Exchange ex = ExchangeBuilder.fanoutExchange(exchange.getId()).durable(true).build();
			declarables.add(ex);
			amqpAdmin.declareExchange(ex);
			log.info("Successfully created exchange {} ({}).", ex.getName(), ex.getType());
		});
		return declarables;
	}

```

Yes, a rabbitmq config can be as complex as that. And that one is not even close to a production grade client with TLS support. 

As a platform developer you would not want your consumers (other teams) to go through that complexity again, potentially making the same mistakes.

You'd rather have them describe a simpler `yml` configuration (placeholders are interpolated by [Ansible](https://docs.ansible.com/ansible/latest/index.html))

```
logging:
  config: {{ platform_home_dir }}/{{ backend_name }}-{{ backend_role }}-logback.xml

spring:
  rabbitmq:
    host: rabbitmq.service.consul
    port: {{ rabbitmq.port }}
    username: {{ rabbitmq.user }}
    password: {{ rabbitmq.password }}

queues:
  exchanges:
    - id: careassist_queues
      routes:
        - id: care_events
          key: care.events
        - id: maintenance_events
          key: maintenance.events
topics:
  exchanges:
    - id: careassist_schedules_topics
      routes:
        - id: schedules
          key: #
```

And use the client like so:

```
/**
 * @author louis.gueye@gmail.com
 */
@Configuration
@Import(PlatformBrokerClientConfiguration.class)
public class PlatformBrokerExampleProducerConfiguration {

	@Bean
	public PlatformBrokerExampleProducerJob platformBrokerExampleProducerJob(final PlatformBrokerClient brokerClient) {
		return new PlatformBrokerExampleProducerJob(brokerClient);
	}
}

```

Tests
===

The `eat your own dog food` principle can easily be implemented by tests.

If your client is difficult to use or configure it's going to show immediately when you try to write a test. You'll know that it's missing features that could greatly ease its adoption by the other teams in the company.
That's the reason why I tend to start with the consumer perspective if I'm not too sure of the direction I should take.

A meaningful test suite can be expansive to build but will payoff in the long run.

Wrap-up
===

This is basically the discipline I try to follow when developing components:
- understand and clarify the requirements
- describe the infrastructure instance in [Terraform](https://www.terraform.io) (instances, security rules, networks, etc)
- since we use [Ansible](https://docs.ansible.com/ansible/latest/index.html) to deploy apps, we need a translator from [Terraform](https://www.terraform.io) output into [Ansible](https://docs.ansible.com/ansible/latest/index.html) input. We chose Nicolas Bering's dynamic inventory to do so.
- we can then write classic [Ansible](https://docs.ansible.com/ansible/latest/index.html) scripts to orchestrate deployments
- write a realistic consumer which acts as the best living documentation
- test your new infra through your consumers

The next episode will put those steps in practice with one of the most foundational component of a platform, namely the discovery component. Stay tuned !

