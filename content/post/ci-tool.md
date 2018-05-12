+++
title = "Changing your CI tool won't fix your broken pipelines"
date = "2018-05-11T20:00:20+02:00"
tags = ['devops', 'ci/cd']
+++

If you think your problem is your CI tool, think again.

Chances are it's just a knee jerk reaction. You probably have a problem with your pipelines.

<!--more-->
![Super safe delivery](https://media.giphy.com/media/11VKF3OwuGHzNe/giphy.gif)


{{%expand "Inspired by: Containers will not fix your broken culture (and other hard truths)" %}}
{{< youtube m2yzEl0EkdY >}}
{{% /expand%}}

<br/>

<p class="small">In this blogpost, I will be focusing on on-prem CI tools (when you host both server and agents); the discussion is slightly different for Cloud CIs (e.g. TravisCI, CodeShip, CircleCI), and I will leave that to another post.</p>


As it's 2018, chances are you all are using some CI tool (e.g. Jenkins, Bamboo, TeamCity, GoCD, ConcourseCI, BuildKite, Spinakker) to run your continuous integration tests (unit, integration, end-to-end, and so on).



Also, you are also probably familiar with the concept of [Deployment Pipelines](https://martinfowler.com/bliki/DeploymentPipeline.html).

<blockquote>
At an abstract level, a deployment pipeline is an automated manifestation of your process for getting software from version control into the hands of your users. Every change to your software goes through a complex process on its way to being released. That process involves building the software, followed by the progress of these builds through multiple stages of testing and deployment. This, in turn, requires collaboration between many individuals, and perhaps several teams. The deployment pipeline models this process, and its incarnation in a continuous integration and release management tool is what allows you to see and control the progress of each change as it moves from version control through various sets of tests and deployments to release to users.
</blockquote>
[Continuous Delivery, THE book](https://martinfowler.com/books/continuousDelivery.html)

Note that your pipeline should be represented in a CI tool. It doesn't mean that your CI tool should define your pipeline.


## Your CI tool is dumb

Some people like to think about the CI tool as the deployment pipeline orchestrator, the tool which will have the definitions of all pipelines and it will automatically trigger a new pipeline chain (a new release candidate) per commit. It will store information about the each release candidate (based on green/red results) and what has been deployed when and where.


While that definition is true, I think it mislead people intro treating it as a magic black box. A sausage factory for production artefacts. In my opinion, that's counterproductive.


I like to say that CI tools are __distributed arbitrary remote code executors__.

__Distributed__ because the server dispatch jobs, and agents/slaves will pick the jobs, execute them locally, and submit results (and sometimes some result files) back to the server.

__Arbitrary__ because the agents will execute _exactly_ the commands defined by the user.


So effectively _your CI tool is just a job scheduler. It will run the commands you define, on the machines you maintain_. That means that a CI tool cannot protect you from yourself.


<p class="small"> Oh, small secret: those nice GUI-oriented tasks on your CI web interface are most likely just wrappers for command line tools installed on the agents.</p>


## Everything is CI

But your CI is a lot more than a CI tool, you actually have a whole ecosystem behind it. For example:

  - CI server
  - CI agents/slaves
      - java
      - maven
      - npm
      - node
      - docker
      - ...
  - Build Dependencies
      - Git repository
      - Maven repository
      - Docker registry
      - Npm repository
  - CI environments (Dev, QA, and so on)

It's not jenkins fault if agents' authentication to nexus is failing; it's not TeamCity fault if your agents don't have connectivity to the DEV environment anymore. You can't blame your CI tool if you didn't provide enough agents and builds are queued. It's not Bamboo fault if some other build left files behind and broke your build - the agent did run _exactly_ the commands it was told to do.

I can tell you this by personal experience: of all the components in CI-land, the CI server is the one least likely to give us trouble. The easy part is maintaining that service running.

## Why there are so many CI tools?

![Delivery cat](https://media.tenor.com/images/b597b9c18690d1960343cc04b7a89e88/tenor.gif)


While all CI tools are job schedulers, they are still different job schedulers.

They can vary on how the pipelines are created (only via code, via web interface, both, some other frankenstein monsters too), how results can be visualised, how easy it is to create new agents on a certain infrastructure provider, what kind of tricks it allows for different processes.

Depending on your CI tool, it might require more code and effort to get to the desired result.

Jenkins is the CI grandapa - it was the king when only CI existed, not really CD. [It doesn't really have the concept of pipeline](https://www.thoughtworks.com/radar/tools/jenkins-as-a-deployment-pipeline). While you can possibly do everything you want, that will require a lot of plugins and a lot of workarounds, with varying amounts of broken. It does require a bit more code and energy to get it to an acceptable state as pipeline tool.

The second wave of CIs (e.g. TeamCity, Bamboo, GoCD) focused a lot more in features like maintainability, CD, pipelines, value-map streams, branch builds and other workflows. Build as code wasn't a big thing.

The third wave of CIs (e.g. Spinakker, BuildKite, ConcourseCI) leveraged build as code and cloud providers.

There's no perfect CI. They all have different features.

{{%expand "So why shouldn't I run the deployments from my machine instead?" %}}
<br/>
It's hard to believe I have to justify this, but it has happened more times I'd like to admit:

  - _Security_: the build agents are not running random javascript pages, cannot target for phishing or social engineering.
  - _Traceability_: there's a correlation between a certain commit, and the exact artefacts it generated
  - _Reproducibility_: all agents (of the same type) should have exactly the same configuration. It doesn't rely on a weird DLL someone installed on their machines back in 2012. CI agents are controlled.
  - _Network privileges_: agents are in controlled and tightly monitored networks; so they usually have permissions to deploy to environments you cannot access from the office networks.
  - _Credentials_: for all the reasons listed before, CI agents have usually more permission to deploy artefacts

If you think you should monkey patch your way to production, I think you should just stop everything you are doing right now and really read [Continuous Delivery](https://martinfowler.com/books/continuousDelivery.html).
{{% /expand%}}

## Should I change my CI tool?

First answer is no. It's very rare to see a single case of a CI tool change that actually delivers what the business case suggested it would.

Changing the CI tool tends to be a big waste of money for the company - and developers are not more satisfied either with their new CI tool. Costs and effort are always underestimated, heavily. Your pipelines are a lot more complicated and interact with a lot more subsystems than people realise. There's a surprising amount of glue code which relies on your CI tool specificities.

Assuming you have a _working_ CI tool, with all your proper pipelines in place, you've already spent some time customising it. A new CI tool means more servers, more ops, more automation, more security assessments, migration, more glue code, learning and adapting every single pipeline you have to new workarounds, authentication, permissions, training, a new learning curve. Those _all_ cost money. Anything you estimate, make it at least double.

The reason a company buys a CI is not out of generosity; it's buying a piece of software which will actually increase developers' and ops' productivity. The CI is not a business deliverable, it's a tool to aid developers and improve quality and speed of the products being delivered.


## Ceci n'est pas une pipe

![Ceci n'est pas une pipe](https://upload.wikimedia.org/wikipedia/en/b/b9/MagrittePipe.jpg)

"Oh yeah, Bamboo, the tool everyone loves to hate."

I have the opinion that as far as you use any CI tool enough, you will hate it. If you don't hate it, maybe you are just not using it enough. Maybe the grass is just greener on the other side. Changing CI tool seems to be quite like hitting your foot with a hammer to forget a headache.

I keep asking myself, why some developers are always so certain that changing the CI tool to some other specific tool is the solution they need?

Is it because CI is such a central tool? Is it because the management wouldn't block the change for once? It is because of all the other CI problems actually manifested themselves on the CI tool screen, of some sort of collective exposure trauma? Is it because people simply don't understand that was broken, and blame the CI tool? <span class="strikethrough">Maybe people want to believe that this new relationship will be totally different, and of course you won't accidentally do exactly the very same mistakes with a new person?</span>

I've seen people migrating to (and from) GoCD because they wanted _every_ single artefact in the company to be part of the same pipeline/stream map. Just because GoCD allows you to do that, it doesn't mean you should - even the Continuous Delivery book tells you it's a very bad idea. Also, it doesn't mean you are forced to do it just because you are in GoCD!

If your life is hell because you tangled all your dependencies, _fix your pipelines_! Don't blame GoCD for your bad pipelines.

Some people went to Jenkins because it's free. Except it's not. The cost to maintain and ongoing customisations it is _far_ more expensive to the company. The cost of CI tool license is peanuts compared to the cost of actually maintaining it. <span class="small">(As we are here, if you use GoCD and have more than 15 people in your company, pay the license fee - to use a proper database).</span>

Are you actually blaming Bamboo because you decided to not use its plan/pipelines the way the tool was designed to be, but rather jobs like Jenkins? Yeah, nah.

You didn't create a pipeline in your CI tool and everything is a mess? How's that caused by your CI tool choice?

## What's your problem?

![What's your problem](https://media.giphy.com/media/OWhTiZgCK3DVe/giphy.gif)

Every CI tool needs customisations (plugins, configurations) to be in a good state. CI tool is not something you just grab it from the shelf and use. It's a pretty flexible and demanding part of the development workflow, and it needs attention.

What's the underlining problem you are trying to address when you want to change the CI tool? Ask that carefully. Chances are the problem is a completely different thing, just being masked as CI tool hate. You should fix the problem you have, not the problem you want to believe you have.

You can critique tools because they incentivise the wrong behaviours. You can also critique tools for being opinionated (and opinions you disagree with).

I do believe that, while there are legit reasons to migrate CI tools, I've never seen them happening in real life. Even Jenkins can serve as a good CI tool, and sometimes it had enough love and care to not be worth migrating.

It's not a matter of what's the best CI tool, but if there's real benefit on migrating. Benefits for the company and for the teams.
