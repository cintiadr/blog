+++
tags = ['secrets', 'aws']
title = "Secrets Management: because we all have secrets to keep"
draft = true
date = "2018-02-09T20:00:20+02:00"
+++

When deploying infrastructure and applications, more frequently than not we have to deal with passwords, API keys, private keys and such. There are several approaches on how to handle them.

<!--more-->

For the purpose of this blogpost, I'll be focusing on secrets required by _machines_ - more specifically for automatically provisioned and maintained ones, as one must.

I believe we have pretty decent human-friendly tools for passwords (namely password managers, they could be even combined with MFA, or several of the tools described here can be used by humans), so I will leave that as an exercise to the reader. Human access to machines (SSH and friends) is a more delicate topic, which I want to cover in another post.


## Let's play utopia

![your secrets are safe with me](http://cdn1.clevver.com/wp-content/uploads/2014/06/pretty-little-liars-alison-your-secrets-safe-with-me.gif)

When handling secrets, there are some broad features desired for secrets management:

  - Encrypted at rest (\*)
  - Encrypted in transit
  - Change auditing (who changed a secret and why)
  - Easy to create new secrets (including _developers_ for app secrets\*\*)
  - Auditable and automated secrets deployments
  - Auditable usage (for each instance using the secret)
  - Reliable (the secret sauce has to be available for the applications when required)
  - Easy rotation, without causing downtime
  - Limited access and opportunity to decrypt or recover secrets
  - Easy to develop applications consuming those secrets
  - They are not printed on the logs

\* _Relying on hard disk encryption here is cheating. Ideally we don't want secrets decrypted on the filesystem at all._

\*\* _If you make it harder on developers to do the right thing, they won't use it. We need to help developers to make the best decision for the business, not become a burden to them._


We aim to have the easiest possible way to deploy and rotate secrets, limiting as much as possible when and where the secrets are in clear text. On the other hand, you want auditability on every step of the way.

But this is real life.
In real life we have to keep the business running, generate profit, have satisfied customers, we have to maintain third party applications.

And maybe the password to JIRA database might not be so risky to the business as passwords to the production databases. We all have our tradeoffs to do, and different parts of your threat model can and should be treated differently.


## It's a big world out there

There are several different tools and strategies to deal with secrets.



Daniel Somerfield has done an amazing talk about secrets, and came up with a classification for different _secrets deployment strategies_:

  - *Orchestrator decryption*: The secrets are encrypted in source code, the orchestrator (Puppet, Chef, Ansible - let me stretch that to CI tools) will deploy the secret, in clear text, to the instances running the applications.
  - *Application decryption*: The secrets are encrypted in source code, the orchestrator will deploy them encrypted. It's the application responsibility to decrypt the secret before using it.
  - *Operational compartmentalisation*: The secrets lifecycle is completely independent of the the application deployment; it's not deployed together with the application, but by a different team/pipeline, and they are just exposed to the applications.



{{< youtube OUSvv2maMYI >}}
<blockquote>At some point we need human intervention, but humans are forgetful, unreliable and prone to be run over by buses.</blockquote>


I want to suggest a broader classification, based on how _the secrets are available to the application_. Those would be *pushed secrets* (when the application has secrets available in clear text) and *pulled secrets*, when the application has to action to retrieve the secrets.

### Pushed Secrets

![Push the penguin](https://media1.tenor.com/images/4a1a5e2c2bb61fdaeaf2e0b912fe9fe5/tenor.gif)

The secrets are readily available for the application in clear text when it starts.
It would typically involve deployments using _orchestrator decryption_ or _operational compartmentalisation_.

This is the highest value you can get if you are currently either not handling secrets in an auditable fashion or if you are commiting secrets in clear text in git.

While the tools do provide some auditing on modification and they are easy to deploy, you still have secrets decrypted at rest and no usage auditability.

#### SCM based tools

There are several popular open source tools in this space. Here a few popular examples:

  - [SOPS](//github.com/mozilla/sops): similarly to hiera-eyaml, only values are encrypted. It works well with GPG\*\*\* and KMS. Supports different file formats. Relies on the fact that the people editing the file always have access to the key, as the file cannot be modified outside sops. Different files can have different keys, and you can have both KMS _and_ GPG on the same file.
  - [git-crypt](//github.com/AGWA/git-crypt): Transparent support in git; encrypts the whole file. Files are kept in clear text locally for those which have access to the private key. Uses asymmetric keys (GPG\*\*\*). Relies on the user using the right convention to avoid secrets being pushed to git upstream unencrypted.
  - [Transcrypt](//github.com/elasticdog/transcrypt): Very similar to git-crypt in behaviour, but uses OpenSSL's symmetric cipher instead of GPG.
  - [Blackbox](//github.com/StackExchange/blackbox): Encrypts the whole file. All files in the repository are encrypted using the same key. Uses asymmetric keys (GPG\*\*\*). Support to be used from puppet as well, like eyaml. It can be possible to never have clear text secrets locally using this tool.
  - [git-secret](//github.com/sobolevn/git-secret): Encrypts the whole file. Uses asymmetric keys (GPG\*\*\*). It's not transparent as git-crypt, as the user has to add files individually.

Usually the CI/CD tool has a build to decrypt and deploy secrets.

We also see quite a few in-house bash/python solutions (using GPG, KMS or any source of keys) or usage of CI environment variables during build time.


#### Orchestrator specific tools

  - [Ansible vault](http://docs.ansible.com/ansible/2.4/vault.html): available for Ansible users. Relies on a symmetric key, so only people with access to the key can modify the file. Distributing the key can be a problem. Encrypts the whole file, and all files in an ansible run should be encrypted using the same key. The secret is decrypted on the machine running ansible, not on the hosts.
  - [Chef Data Bags](//docs.chef.io/secrets.html) and [Chef vault](//github.com/chef/chef-vault): available for Chef users. Uses symmetric key, just like ansible. Encrypts the whole file. The secret is decrypted in the node only, and they should have the shared secret. Chef vault restricts which nodes can decrypt a certain Data Bag, using the Chef client key.
  - [hiera-eyaml](//github.com/voxpupuli/hiera-eyaml): while tailored for puppet, eyaml (and hiera) can be used independently. Only supports yaml files. Uses asymmetric keys (GPG\*\*\* and PKCS7), gpg-agent [appears supported](//lzone.de/blog/Hiera+EYAML+GPG+Troubleshooting) only when running outside puppet server. When used with puppet, the secrets will be decrypted by the puppet server. Only encrypts values, and allows people without the private key to edit the file (and even add encrypted data). It supports GPG key ring, with multiple private keys allowed to decrypted the secrets.
  - Blackbox for puppet(see above)


Please note that [Docker Secrets](//docs.docker.com/engine/swarm/secrets/) and [Kubernetes Secrets](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/auth/secrets.md) are *not* methods for storing secrets. They only provide a secure way for the _cluster managers_ to share secrets with the containers. They absolutely don't solve the problem of how to get the secrets to the cluster in the first place, and at most they aid _operational compartmentalisation_ deployment for secrets.
Note that, as of today, there's no control for secret access in docker swarm (a newly created container can be deployed to read _all_ the secrets available on the cluster). Also, there's no versioning concept for secrets yet nor audit logs. I do expect they are working to address that.

\*\*\* _gpg-agent is required to have private keys with passphrase. Because of the way GPG works, it's possible to have multiple private keys allowed to open the same file (as far as they are all included as recipients). All tools mentioned here appear to be taking advantage of that. And before I get any GNU nerd talking about PGP vs GPG, just don't.


#### Bootstrapping

Regardless of which tool you decide to use, you still need to solve their bootstrap; the machine which will decrypt the secret needs to have a certain private key or shared secret. And you might even want to rotate those keys every so often.

Because of that, it's common to use at least two of these tools.

There's no easy way of solving this, and usually the solution is either human intervention (e.g. secrets are stored in another secure location and copied manually when the puppet server starts) or some other sort of trusting mechanism (e.g. using IAM roles).

Access Management to secrets is trusted to the orchestrator, which by definition has access to all secrets and deploy them only to registered and checked nodes.

### Pulled Secrets

![cats and milk](https://media.giphy.com/media/B6ZOD3aNT3Lxe/giphy.gif)

The secrets are available for the application either encrypted or needs to be retrieved from somewhere external to the machine running it.
It would typically involve _application decryption_ and _operational compartmentalisation_ deployment strategy, or an external secret management tools.

This category have a much higher barrier of entry, as they require all the relevant applications to be changed and be actively responsible for secrets management.

But it has very interesting features:
  - Encryption at rest (no secrets written to the filesystem)
  - Usage audits
  - Depending on implementation, ephemeral credentials can be available without restart

Cons:
  - Operationally expensive to maintain and use secrets
  - Single point of failure

This application/appliance needs to be extremely reliable and high available, otherwise your whole production will be all down. The ongoing maintenance cost is much higher.

The deployments are more complicated by definition. The biggest tendency is to create big silos, those who create credentials and those who consume them (while no tools enforce that, that's the natural tendency for this type of tool due to the higher complexity involved when creating and deploying secrets).

As using these tools requires a lot more effort than the previous class of tools, do not take this decision lightly.

#### Tools

Some examples:
  - [KeyWhiz](//square.github.io/keywhiz/): it's a key management system and secret management system.
  - [Knox](//github.com/pinterest/knox/wiki)
  - [CredStash](https://github.com/fugue/credstash),[Biscuit](https://github.com/dcoker/biscuit), [Sneaker](https://github.com/codahale/sneaker): tools using KMS as the encryption backend.
  - [AWS SSM Parameter store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-paramstore.html): allow AWS EC2 and ECS instances to request specific secrets.
  - [Vault](https://www.vaultproject.io/): it's a key management system, access management system, secret management system, and certificate authority. It would replace AWS IAM/STS, AWS KMS, AWS SSM Parameter Store (AWS Certification Manager can't generate certificates so easily for internal servers yet). HA is officially supported only on consul clusters.


The tools in this space have a lot more features, including auditability and node authentication. A good read on them can be done in:
<https://gist.github.com/maxvt/bb49a6c7243163b8120625fc8ae3f3cd>

It's common as well to see applications communicating straight to HSM to handle decryption, but the effort and price could be prohibitive. The logs and configuration for HSM tend to not be so readily accessible, and error messages are pretty cryptic.

I've also seen some bad ideas about putting a 'proxy' to inject all credentials needed for the requests. Based on the fact that you'd need to implement the authentication, authorization per service per secret, make sure it's high available and extremely protected, and you still need to provide an automatic way deploy secrets to this server from code, I'd just suggest you don't reinvent the wheel and use use one of the tools that already exist.

#### Lightweight pulled secrets
Due to the nature of security tools, sometimes teams implement complex tools without a lot of appreciation for the security outcome expected. Some teams implement AWS SSM/HSM/Vault, but the secrets are decrypted during boot time (or docker container initialisation), and write it in plain text to the disk.

Effectively, you just lost all the advantages of these tools (auditability on when a secret was needed, ability to rotate keys without downtime, and not write clear text to disk), but you still have to handle the maintenance cost. IMO, this class of tooling should only be used if you decide to go all the way in.

#### Bootstrapping

Note that none of the tools mentioned here handle how the secrets end up in the external secret management system; usually that tends to be one of the _SCM_ tools described before.

Also, the nodes have to authenticate with the secret management system. There are several ways of doing it, via certificate, IAM roles, depending on the tool.


## Credential-less applications

![Password](http://it-positive.co.uk/wp-content/uploads/2017/03/passwordeasy.jpg)

You are probably now thinking this is just too hard, can you just get rid of secrets altogether?
Sometimes you can. Sometimes your access management system (e.g. AWS IAM roles, security groups) can be enough for some services. IAM role is extremely powerful as we don't run into bootstrap problems.

You can achieve similar effects using Vault on premises for certain services, but you are left to solve how to bootstrap the clients to retrieve passwords from Vault.

But there will _always_ be some secrets. I think it would be naive to believe otherwise.


## This is fine

![this is fine dog](https://pbs.twimg.com/media/C3gIjDSWcAAQxoz.jpg)

Secrets are bad, but we just have to deal with the fact that they do exist.

There are several tools (open source and proprietary) that can fit some parts of your workflow, but you need to find the right balance for each case. It's fine and expected to use multiple tools at the same time, as far as they complement each other for your use case.

Don't commit clear text secrets to repositories. You are a single git clone away of bad things.

Don't go overboard and install vault + consul cluster + deployment if you don't absolutely need the detailed usa auditing or if you are not willing the change the applications to use it. If you are in AWS, ask yourself what exactly are you getting out of it that is better for the company then AWS IAM/STS + SSM Parameter Store + KMS + Cloudwatch.

Be just as paranoid as needed.  
