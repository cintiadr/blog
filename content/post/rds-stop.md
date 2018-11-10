+++
tags = ['rds', 'aws']
title = "How much time does it take to stop an AWS RDS?"
draft = false
date = "2018-11-09T20:00:00+11:00"
featured_image = "https://gph.is/1kGUxZ9"
+++

In case you missed the announcement, AWS now allows [stopping and starting multi-az RDS](https://aws.amazon.com/about-aws/whats-new/2018/10/amazon-rds-stop-and-start-of-multi-az-instances/). That's a pretty great feature!


Do you know how much time does it take to stop an RDS instance?

<!--more-->
![oh, stop it](https://media.giphy.com/media/oFeUVZfiuim9G/giphy.gif)

As you'd expect the answer is (of course): it depends.

Until last week, every single attempt I did to stop an instance took less than 20 minutes, regardless of database size or instance type. Maybe I was lucky.

Recently we came across a case where we requested a multi-az RDS to be stopped via AWS console - without any snapshot. It's a database with a few terabytes of data, and we had plans to have it running again soon.

The instance entered status 'Stopping', and after one hour there was no sign it would finish any time soon. A quick [google search](https://medium.com/insping/how-much-time-can-aws-take-to-stop-a-rds-database-instance-1bb7c67752b2) showed me that clearly I wasn't the first case of something like that. It appeared to me that the RDS was 'stuck' on that state, we couldn't cancel or do anything other than wait. There was no obvious snapshot being taken, and we just hoped that AWS support would 'un-stuck' the RDS.


![I refuse to stop](https://media.giphy.com/media/SiCZLMW9HGC7VfMvPe/giphy.gif)

Turns out that after 6 hours after I attempted to stop the instance, the instance was successfully stopped without any interference from AWS.

Support then explained to us that, when you stop an RDS, it will _always_ create a DB snapshot. That snapshot is hidden from us, it's not visible and it's done to improve data durability. The time the snapshot will take depends on IOPS and number of changes since the last snapshot taken (snapshots are supposedly incremental).

Why our snapshot took 6 hours is not really clear to me. We had an automatic snapshot taken 30 minutes before I attempted to stop the instance, and I was surprised we managed to write so much data in such small interval. But that's as good as it gets, I suppose.

![Stop it](https://media.giphy.com/media/3o6ozsZE22s7LaRfAQ/giphy.gif)

My advice? Assume it can take hours, because that can be the case. 
