+++
tags = ['devops', 'ci/cd']
title = "Build and infrastructure should not be 'code'"
draft = true
date = "2018-02-21T20:00:00+11:00"
+++

You probably hear the expression 'build as code' and 'infrastructure as code', and I must admit
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
Instead of applying manual changes to create and maintain snowflake infrastructure, you change files under source control, you can run tests, you deploy using a CI/CD tool, you have full auditability and control.

All the advantages that development has had for the last decades, now applied to infrastructure.
Nothing short of a miracle, a devops miracle.

There are numerous texts about the advantages of DevOps and infra as code, and I don't need to repeat them here.

Builds-as-code or pipelines-as-code follow same principle. Your CI/CD pipeline should be defined in a file, commited, and not manually modified.

## The state of infrastructure

One of my favourites tools in DevOps-land is [terraform](https://www.terraform.io/).
It allows us to define and update our
_whole_ infrastructure using a simple domain specific language (DSL).

You can see in [Github](https://github.com/openmrs/openmrs-contrib-itsm-terraform/blob/master/ako/main.tf),
I'm using it to define all my networks, my VMs, security groups, DNS names
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

Each cloud provider has their own vendor-specific tool to define infrastructure state (e.g. Cloudformation for AWS). Some use YAML/json, some use DSL. You change the files, you run the tool, and voilà, the infrastructure is updated.

But look closer.
Do you see that none of those files are _actually_ code????

You are not writing objects, while, for, if, else, exceptions.
You are not sending messages to queues. You are not programming a framework, connecting to the database, interacting with third party systems.

They are not functional, imperative or object-oriented languages. They are a much simpler language to define state. They are much closer to configuration files than code.

<br/>
They are **state** files. **State files**.
<br/>

You are defining the state that your infrastructure should be. And the tool will calculate the differences, and act on them. Rerunning the tool will have no effect, as there are no deltas to be applied.

<br/>
Let's show you a bad example:

{{< highlight bash >}}

#!/bin/bash -eux

aws ec2 run-instances --image-id ami-xxxxxxxx --count 1 \
  --instance-type t2.micro --key-name MyKeyPair \
  --security-group-ids sg-xxxxxxxx --subnet-id subnet-xxxxxxxx

{{< / highlight >}}

If you rerun this bash script, you end up with... a very confusing infrastructure. And then you have to recreate some sort of poor-man's-terraform in bash to avoid duplicates.

You don't want infrastructure as code.

You want __infrastructure as state__. You want the files committed to git to _actually_ reflect what's in production, as of a [map of the territory](https://wiki.lesswrong.com/wiki/The_map_is_not_the_territory), giving good visibility and predictability of your whole infrastructure.

And you want a tool which can read the state files and act based on them.


## The map is not the territory

But your infrastructure is _more_ than your state files.

When you think about production infrastructure, you think about a very specific set of resources: a couple of DNS entries, a few databases, a few VMs clusters with certain configuration. While some of this resources have auto-healing, they are still a collection of live resources. While you don't specifically care about their ID, and you can recreate them, they do still exist.

Your staging infrastructure will not have the same resources as the production infrastructure. They can be created using the same state files, but you know that the production database _is not_ the staging database.

Let me take this opportunity to explain some concepts as I see them.

![bye bridge](https://media.giphy.com/media/B6chryYJDMaLC/giphy.gif)

A __stack__ is a collection of live infrastructure sources. You can have, for example, the _production database stack_, a collection of all infrastructure resources related to the database in production.
This concept correlates closed to terraform _state file_ and Cloudformation stacks.

A _stack_ is maintained by state files, and the collection of state files for a certain stack is called __stack definition__.
The _stack definition_ should be fully committed to SCM. Some other approaches generates parts of the stack definition during the first steps of the CD build (e.g. calculate latest AMI available).

The _stack definition_ is formed by two parts: __templates__ (resources definition) and __configuration__.

In terraform lingo, _configuration_ would be typically done by its powerful variables in _.tfvar_ files (in combination with workspace); _templates_ would be all resources from .tf files.

In Cloudformation, _configuration_ would be parameters (using a tool like  [cfn-sphere](https://github.com/cfn-sphere/cfn-sphere) or [troposphere](https://github.com/cloudtools/troposphere) reading [configuration files](https://github.com/amosshapira/thermal/tree/master/cloudformation)).  

You want the use the same _templates_ across all environments;
the _configuration_ will allow the little differences between environments
(different AMIs, bigger VMs, CIDR range, network restrictions, so on).

I've never seen a company where production was identical to the other environments;
hence, the stack definitions are different by design.
You always want the configuration differences to be as minimal as possible and practical.

I've seen people using both monorepos (all stack definitions in the same SCM repository) and even
one stack definition per repository. I do personally prefer one repository per team maintaining it.

## Deployment

![On fire](https://media.giphy.com/media/MdkEfo85Ejcze/giphy.gif)

Deploying infrastructure is not the same as deploying applications. And that's where again the analogy 'something-as-code' fails us miserably again.

Let's think about application deployments.
During the early stages of the deployment pipeline, the application will be packaged _once_.
This very same package will be copied to _all environments_ during the CD deployment pipeline.   

It's so concrete it's almost touchable. If version 1.0-123 is deployed to development, the very same file with the very same checksum will be deployed to production. If you ssh into the production environment, the build artefact will be there - running. There's a direct correlation between what was generated during build time and what is running in production.

Except for databases schema changes, rolling back an application is a matter of replacing a couple of files and restarting it.

Configuration files for applications very rarely change, and mostly relate to external interfaces (database and cache connections, debug logs, so on).

<br/>
![fly fly fly](https://media.giphy.com/media/hVIupGTXnWqDC/giphy.gif)

Infrastructure deployment, on the other hand, is _updating infrastructure stacks_ using stack definitions as a medium. The stack definition itself is useless after deployment, it's just a good abstraction language.

Infrastructure stacks vary a lot more per environment compared to applications, due to the very nature of
'production-like' environments.
It's extremely common to have infrastructure changes that should only affect a couple of environments
(e.g. only dev and qa can be accessed from the office), or even common resources have infrastructure
(e.g. management or CI infrastructure).

And that's why I never fully embraced the concept of a single infrastructure pipeline for all environments sharing the same template.

I do prefer instead to have one pipeline per stack. The early stages of the build will calculate if there are changes to be applied (and notify via comment or notification).

Because configuration has such big impacts on the stack definitions, it can be used as a tool for phased
deployments. 


<br/>

Just check the language we use for infrastructure resources and applications:

" The production database is down"

" The poodle application is down in production"


### Development



### Testing
