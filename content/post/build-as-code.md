+++
tags = ['devops', 'ci/cd']
title = "Build and infrastructure should not be 'code'"
draft = false
date = "2018-03-03T20:00:00+11:00"
+++

You've probably heard the expressions 'build as code' and 'infrastructure as code', and I must admit
they are my pet peeves.

They are terribly misnamed, and I'm sure I'm not the only one thinking like that.

<!--more-->
![building stuff](https://media.giphy.com/media/c5eqVJN7oNLTq/giphy.gif)

I know this is a pretty polemic topic, but here I go.

## New phone who dis

Maybe you are living under a rock and you don't know what I'm talking about.
[Martin Fowler](https://martinfowler.com/bliki/InfrastructureAsCode.html) wrote a bliki
entry about infra-as-code, and I suppose it's as mainstream as it can get.

<blockquote>Infrastructure as code is the approach to defining computing and network infrastructure through source code that can then be treated just like any software system. Such code can be kept in source control to allow auditability and ReproducibleBuilds, subject to testing practices, and the full discipline of ContinuousDelivery. </blockquote>

The idea is great.
Instead of applying manual changes to create and maintain snowflake infrastructure, you now change files under source control.

All the advantages that SCM has brought to development, now applied to infrastructure. Run tests, deploy using a CI/CD tool, full auditability and control.
Nothing short of a miracle, a devops miracle.

There are numerous texts about the advantages of DevOps and infra as code, and I don't need to repeat them here.

Builds-as-code or pipelines-as-code follow same principle for CI/CD pipelines.

## The state of infrastructure

One of my favourites tools in DevOps-land is [terraform](https://www.terraform.io/).
It allows us to define and update our
_whole_ infrastructure using a simple domain specific language (DSL).

You can see this example in [Github](https://github.com/openmrs/openmrs-contrib-itsm-terraform/blob/master/ako/main.tf),
I'm using terraform to define all networks, VMs, security groups, DNS names
(that's the infrastructure I maintain for
[OpenMRS](https://openmrs.org/) Community).

{{< highlight dsl >}}
  module "single-machine" {
    source            = "../modules/single-machine"

    # Change values in variables.tf file instead
    flavor                = "${var.flavor}"
    hostname              = "${var.hostname}"
    region                = "${var.region}"
    update_os             = "${var.update_os}"
    use_ansible           = "${var.use_ansible}"
    ansible_inventory     = "${var.ansible_inventory}"
    has_data_volume       = "${var.has_data_volume}"
    data_volume_size      = "${var.data_volume_size}"
    has_backup            = "${var.has_backup}"
    dns_cnames            = "${var.dns_cnames}"
    extra_security_groups = ["${openstack_networking_secgroup_v2.secgroup_ldap.name}"]

    ...
  }

  resource "openstack_networking_secgroup_v2" "secgroup_ldap" {
    name                  = "${var.project_name}-ldap-clients"
    description           = "Allow ldap clients to connect to server (terraform)"
  }
{{< / highlight >}}

<br/>
Each cloud provider has their own vendor-specific tool to define infrastructure state (e.g. Cloudformation for AWS). For the purpose of this blogpost, I will call them collectively __infra tools__.

Some of the tools require you to write YAML/json files, some use DSL. You change the files, you run the _infra tool_, and voilà, the infrastructure is updated to match the files.

But look closer.
Do you see that none of those files are _actually_ code????

You are not writing objects, while, for, if, else, exceptions.
You are not sending messages to queues. You are not programming a framework, connecting to the database, interacting with third party systems.

They are not functional, imperative or object-oriented languages. They are a much simpler language to define state. They look like a lot more like configuration than code.

<br/>
They are **state** files. **State files**.
<br/>

You are defining the state that your infrastructure should be. And the _infra tool_ will calculate the differences, and act on them. Rerunning the tool will have no side effect, as there are no deltas to be applied.

<br/>
Let's show you a __bad__ example:

{{< highlight bash >}}

#!/bin/bash -eux

aws ec2 run-instances --image-id ami-xxxxxxxx --count 1 \
  --instance-type t2.micro --key-name MyKeyPair \
  --security-group-ids sg-xxxxxxxx --subnet-id subnet-xxxxxxxx

{{< / highlight >}}

If you rerun this bash script, you end up with... a very confusing infrastructure. And then you have to recreate some sort of poor-man's-infra-tool to avoid duplicates.

You don't want infrastructure as code.

You want __infrastructure as state__. You want the files committed to git to _actually_ reflect what's in production, as of a [map of the territory](https://wiki.lesswrong.com/wiki/The_map_is_not_the_territory), giving amazing visibility and predictability of your whole infrastructure.

And you want an _infra tool_ which can read the state files and act based on them.


## The map is not the territory

But your infrastructure is _more_ than your state files.

When you think about production infrastructure, you think about a very specific set of resources: a couple of DNS entries, a few databases, a few VMs clusters with certain configuration. While some of this resources are auto-healing, they are still a collection of live resources. While you don't care about their specifically ID, and you can recreate them, they do exist. Some of these resources have state or data, and some of them will cause downtime if a recreation is needed.

Your staging infrastructure will not have the same resources as the production infrastructure. They can be created using the same state files, but you know that the production database _is not_ the staging database.

Let me take this opportunity to explain some concepts as I see them.

![bye bridge](https://media.giphy.com/media/B6chryYJDMaLC/giphy.gif)

A __stack__ is a collection of _live_ infrastructure resources. You can have, for example, the _production database stack_, which will have a database cluster, a few DNS entries, sometimes some bits of networking and security. _Stack_ is a logical group of infrastructure resources currently running.
This concept correlates closed to terraform _state file_ and Cloudformation stacks.

A _stack_ is maintained by state files, and the collection of state files for a certain stack is called __stack definition__.
The _stack definition_ should be fully committed to SCM. Some other approaches generates parts of the stack definition during the first steps of the CD build (e.g. calculate latest AMI available).

The _stack definition_ is formed by two parts: __templates__ (resources definition) and __configuration__.

In terraform lingo, _configuration_ would be typically done by its powerful variables in _.tfvar_ files (in combination with workspace); _templates_ would be all resources from .tf files.

In Cloudformation, _configuration_ would be parameters (using a tool like  [cfn-sphere](https://github.com/cfn-sphere/cfn-sphere) or [troposphere](https://github.com/cloudtools/troposphere) reading [configuration files](https://github.com/amosshapira/thermal/tree/master/cloudformation)).  

You want the use the same _templates_ across all environments;
the _configuration_ will allow the differences between environments
(different AMIs, bigger VMs, CIDR range, network restrictions, so on).

I've never seen a company where production was identical to the other environments;
hence, the stack definitions are different by design.
You always want the configuration differences to be as minimal as possible and practical.

I've seen people using both monorepos (all stack definitions in the same SCM repository) and even
one stack definition per repository. I do personally prefer one repository per team maintaining it.

### Deployment

![On fire](https://media.giphy.com/media/MdkEfo85Ejcze/giphy.gif)

Deploying infrastructure is not the same as deploying applications. And that's where again the analogy 'something-as-code' fails us miserably.

<br/>

Just check the language we use for infrastructure resources and applications:

"The production database is down"

"The poodle application is down in production"

We intuitively know they are not the same.  

<br/>

Let's think about application deployments.
During the early stages of the deployment pipeline, the application will be packaged _once_.
This very same package will be copied to _all environments_ during the CD deployment pipeline.   

It's so concrete it's almost touchable. If version 1.0-123 is deployed to development, the very same files with the very same checksum will be deployed to production. If you ssh into the production environment, the build artefacts will be there - hopefully running. There's a direct and concrete correlation
between what was generated during build time and what is running inside production.

Configuration files for applications very rarely change, and mostly relate to external interfaces (database and cache connections, debug logs, so on).

<br/>
![fly fly fly](https://media.giphy.com/media/hVIupGTXnWqDC/giphy.gif)

Infrastructure deployment, on the other hand, is _updating infrastructure stacks_ using stack definitions as a medium. The stack definition itself is useless after deployment, it's just a good abstraction language.

Infrastructure stacks vary a lot more per environment compared to applications, due to the very nature of
'production-like' environments.
It's extremely common to have infrastructure changes that should only affect a couple of environments
(e.g. only dev and qa can be accessed from the office), or even common resources have infrastructure
(e.g. monitoring servers, issue tracker, servers for orchestrators).

<br/>

And that's why I never fully embraced the concept of a single infrastructure pipeline for all environments sharing the same template.
I do prefer instead to have one pipeline per stack. The early stages of the build will always be triggered
and will calculate if there are changes to be applied (and notify via comment or notification),
so we always aware of changes pending to be deployed to a stack.
A deployment is a deliberate decision, and it's only done after the person analyses the changes and approves them, per stack.

Contrary to what some people believe, just because different stacks share the same template files
they don't _need_ to be on the same deployment pipeline. It's something you can totally do if you want and you see any advantage, but remember that configuration will cause the stack definition to be different
for each environment.

I still do prefer the ability to enable each change per environment based on configurations
instead (sort of feature flags disabling new resources), allowing different pace for different changes.


As far you control the rollout of the new changes to all relevant environments in some other way
(let's say, JIRA tickets that are only closed when all those environments are deployed),
you should be fine.


### Testing

A huge advantage of using stack definitions is that now one could easily recreate and modify stacks.

This opens a whole new world for validating (naming conventions, tags conventions, static security checks) and testing
(create a new stack and check for a health check on a certain URL).

But it's important to note that _you are not here to test the infra tool_. You don't need to test that terraform
or Cloudformation work as they should on each deployment terraform and Cloudformation _will_ create the resources you asked.

The question for your integration tests is: does that infrastructure _actually_ delivers what you need? Did your change break connectivity to the application you need? Think holistically about your stack and infrastructure when creating your tests.

Leave tool tests for the tools themselves. Terraform is open source, so you can spend your time creating pull requests with tests for them if you really feel like.


### Local development

![coding coding](https://media.giphy.com/media/xT8qB2HYA1vVSxooSY/giphy.gif)

Most infra tools will allow you to do some pre-validation and check differences what would be applied, but do not think for a moment this is enough.

That would be pretty much equivalent to say that if it compiles, it will run successfully.

Sometimes the stack is isolated enough a person can create a new one to test the new infrastructure changes. Sometimes that's not possible, due to dependencies on other stacks or network complications; for those cases, there's usually a designated environment (e.g. sandbox environment) that is considered to be less risky and can be broken for a couple of minutes if something doesn't go according to plan.

Not having the possibility of deploying the stacks locally (even for a sandbox environment) is a pain; and makes the development much slower (as every change needs to be committed and reviewed before you even had the opportunity to run it).

I never saw a single developer been told to deploy their changes to dev without testing locally, without the ability to at least run some integration or manual tests. We know that would slow down development a lot, and cause code to be as conservative as possible to work not more efficiently, but on less attempts. Do not cause that to your infrastructure.

## Conclusion

![Going](https://media.giphy.com/media/kZD8cN1MycfKw/giphy.gif)

Infrastructure is not code.

Infrastructure is something that should be modified by an infra tool, using state files.

Stop treating infrastructure changes like application changes. They are not the same, while some principles still apply.

Testing of infrastructure is an amazing possibility, but you want carefully consider what needs to be tested.
