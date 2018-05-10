+++
title = "Changing your CI tool won't fix your broken pipelines"
date = "2018-05-09T20:00:20+02:00"
tags = ['devops', 'ci/cd']
draft = true
+++

If you think your problem is your CI tool, think again.

You are probably just having a knee jerk reaction of a much more profound problem.<!--more-->

<br/>

Bridget Kromhout did a good talk in a similar theme, "Containers will not fix your broken culture (and other hard truths)".

{{< youtube m2yzEl0EkdY >}}

<br/>

As it's 2018, chances are you all are using some CI tool (e.g. Jenkins, Bamboo, TeamCity, GoCD, TravisCI, CircleCI...) to run your continuous integration tests (unit, integration, end-to-end, and so on).

Also, you are as well familiar with the concept of [Deployment Pipelines](https://martinfowler.com/bliki/DeploymentPipeline.html). A pipeline is the path an artefact will go through since the commit to source control all the way to your customers - the closer it gets to production, the more confidence you have of its quality and suitability to be deployed.

## CI like there's no tomorrow

Some people like to think about the CI tool as the pipeline orchestrator, the tool which will have the definitions of the pipelines and it will automatically trigger a new instance (or a new release candidate). Which is a valid way of looking at it. 



## Ceci n'est pas une pipe

But even then, your CI tool _is not_ your pipeline. Deployment pipelines are first and foremost an abstraction. After the team decides how the pipeline should be, it can (and should) be represented on your favourite CI tool - sometimes with some workarounds. I will keep my word: your pipeline should _never_ be radically affected by your CI tool.
