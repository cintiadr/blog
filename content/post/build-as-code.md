+++
tags = ['devops', 'ci/cd']
title = "Build and infrastructure should not be 'code'"
draft = true
date = "2018-02-21T20:00:00+11:00"
+++

You probably hear the expression 'build as code' and 'infrastructure as code', and I must admit
they are my pet peeves!

They are terribly misnamed, and I'm sure I'm not the only one thinking like that.

<!--more-->
![building stuff](https://media.giphy.com/media/c5eqVJN7oNLTq/giphy.gif)

## New phone who dis

Maybe you are living under a rock and you don't know what I'm talking about.
[Martin Fowler](https://martinfowler.com/bliki/InfrastructureAsCode.html) wrote a bliki
entry about infra-as-code, and I suppose it's as mainstream as it can get.

<blockquote>Infrastructure as code is the approach to defining computing and network infrastructure through source code that can then be treated just like any software system. Such code can be kept in source control to allow auditability and ReproducibleBuilds, subject to testing practices, and the full discipline of ContinuousDelivery. </blockquote>

The idea is great.
Instead of applying manual changes to create and maintain snowflake infrastructure, you change files under source control, you can run tests, you deploy using a CI/CD tool, you have full auditability and control.

All the advantages that development has had for the last decades, now applied to infrastructure.
Nothing short of a miracle, a devops miracle.

There are numerous texts about the advantages of DevOps and infra as code, and I don't need to repeat it.

Builds-as-code or pipelines-as-code follow same principle. Your CI/CD pipeline should be defined in a file, commited, and not manually modified.

## The same until it isn't

One of my favourites tools in DevOps-land is [terraform](https://www.terraform.io/).
It literally allows us to define and update our
_whole_ infrastructure using a simple domain specific language (DSL).

You can see in [Github](https://github.com/openmrs/openmrs-contrib-itsm-terraform/blob/master/ako/main.tf),
I'm using it to define all my networks, my VMs, security groups, DNS names.
That's a real and live repository. That's the infrastructure I maintain for
[OpenMRS](https://openmrs.org/) Community, it's not a hello-world example.

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
{{< / highlight >}}

Each cloud provider has their own tool to define infrastructure state (e.g. Cloudformation for AWS). Some use YAML/json, some use their DSL. You change the files, you run the tool, and voilá, the infrastructure is updated.

But come closer.

Do you see that none of those files are _actually_ code????

They are **state** files. **State files**.

<br/>
You are not writing objects, while, for, if, else, exceptions, transform.
You are not writing messages. You are not programming a framework, connecting to the database, interacting
with third party systems.

They are not functional, imperative or OO languages. They are a language to define state, or somehow configuration.

You are defining the state that your infrastructure should look like. And the tool will act based on that.


## Tomato, tomato
