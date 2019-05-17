+++
tags = ['RDS', 'AWS', 'EBS']
title = "Why is my RDS/EBS volume slow after restoring from a snapshot?"
draft = false
date = "2019-05-17T20:00:00+10:00"
+++


_Certainly the cloud is going to solve all our problems._

![Dog on top of a turtle walking around](https://media.giphy.com/media/B2vBunhgt9Pc4/giphy.gif)


Recently, we came across a very unusual scenario.

My team needed a replica from production to run some performance tests.
We took a snapshot and created another identical RDS instance based on said snapshot.


After the new RDS was available, we attempted to connect our application to it.
To our surprise, the copied RDS was extremely slow; big queries that took less than 5 seconds in production were taking more than 60 seconds (or even timing out).

<!--more-->


We checked everything. Instance type. Parameter groups. Performance insights. What was possibly wrong?

As we discussed, a colleague mentioned that it wasn't the first time they'd heard about that. They had a feeling that a fresh RDS (with production data) needed to 'warm up' somehow for a few hours before being usable performance-wise.


That didn't make sense, I thought.
I found it hard to believe that PostgreSQL indexes (and other data used for queries planning) were lost when recovering the RDS from a snapshot.
Seemed too unlikely, clearly we were doing something wrong - so I decided to ask google.  

I found an obscure question in [stackoverflow](https://stackoverflow.com/questions/47545414/aws-rds-instance-created-from-snapshot-very-slow) mentioning that - when creating a new RDS from a snapshot - the data is only downloaded from S3 when needed. It seemed a little bit odd.

----------------

There's nothing specific about RDS in AWS docs. But there's [this](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-restoring-volume.html) about EBS volumes created from snapshots:

>    New volumes created from existing EBS snapshots load lazily in the background. This means that after a volume is created from a snapshot, there is no need to wait for all of the data to transfer from Amazon S3 to your EBS volume before your attached instance can start accessing the volume and all its data. If your instance accesses data that hasn't yet been loaded, the volume immediately downloads the requested data from Amazon S3, and continues loading the rest of the data in the background.
>
>
>    ...
>
>    New EBS volumes receive their maximum performance the moment that they are available and do not require initialization (formerly known as pre-warming). However, storage blocks on volumes that were restored from snapshots must be initialized (pulled down from Amazon S3 and written to the volume) before you can access the block. This preliminary action takes time and can cause a significant increase in the latency of an I/O operation the first time each block is accessed. Performance is restored after the data is accessed once.
>
>    For most applications, amortizing the initialization cost over the lifetime of the volume is acceptable. To ensure that your restored volume always functions at peak capacity in production, you can force the immediate initialization of the entire volume using dd or fio.

<br/>

Firstly, wait what? It sounds equally awesome and scary. So you told me my data was on the disk when it wasn't? Ok, then.


While we can safely assume that an RDS is backed by EBS volumes (and an RDS snapshot is probably an EBS snapshot), I just couldn't believe they kept the same behaviour for RDS.
Clearly going to S3 when running a PostgreSQL query didn't sound like a great idea.

So I raised a support ticket.

----------------

Guess their answer.

![Surprised turtle](https://media.giphy.com/media/xm27OEVjSxGh2/giphy.gif)

>   You correctly mentioned, new volumes created from existing EBS snapshots load lazily in the background.
>
>    This means that after a volume is created from a snapshot, there is no need to wait for all the data to transfer from Amazon S3 to your EBS volume before your attached instance can start accessing the volume and all its data.
>
>    Suggestions for your issue:
>
> 1) You can manually trigger a vacuum analyze, this will do a full table scan of each table within scope to update the planner with fresh statistics.
>
> 2) If number of tables are less, you can run "select * from table-name" for all the tables. This will bring the data into EBS volume from S3.
>
> 3) If you want to bring whole data in one go, then you can use pg_dump of PostgreSQL.
>
>
> (...)
>
> When new volumes are created from existing EBS snapshots then data is loaded from S3 to the new EBS volume which is lazy loading so it takes time. It is slow for the first time only, once the data is loaded, you will not face slowness. So, it would be better if you trigger vacuum analyze or run select command on tables as that would bring all the tables into new EBS volume.


I also confirmed with support that it affects point-in-time recoveries as well.
Changing instance size or storage type doesn't cause the same behaviour.

Also there's no way for us to tell how much data is actually downloaded to the disk (or how much is left in S3).

----------------

I'm not entirely sure what to think about it, but clearly that's something I needed to know.

![turtle light sabers fight](https://media.giphy.com/media/ys2SDO6BgXjvG/giphy.gif)
