+++
title = "Ldap"
date = "2018-02-07T20:00:20+02:00"
tags = ['ldap']
draft = true
+++


Andrew Fong, Infrastructure Manager at Dropbox:

Operationally mature cultures are ones that are able to understand the tradeoffs that they are making in a production environment and the impact that has to the business.

Pagerduty blogpost




The blogpost mentioned above is good, but it's intentionally vague.

While it's not hard to understand what is an operational mature company, everyone seems to have a different opinion on what it would mean for a development team. I'm explicitly going to avoid the


I have this belief in me that being 'operational mature' for product teams has no requirement on conventional infrastructure skills. It's a mindset, and has no relationship with knowing AWS or terraform, and more of applied systems theory.


Let me try to explain how I see that applied to our development teams:

They think customer first; they correctly identify incidents and are not afraid to declare one. They jump to fix incidents and understand that they don't necessary have all the tools to fix it, but they are ultimately responsible to ensure the fix (and comms) are put in place in a timely manner;
They don't believe in 'human errors'; but rather, they understand that a situation mislead a person to an undesired outcome. Postmortems are sincere; people have no trouble explaining why they took a certain decision because they know that will improve production going forward;
They understand that alarms and incidents are a proxy for customer pain;
Postmortem are encouraged and part of the culture; RCA are identified, and actions are raised and acted upon, as they know it will improve customers' experience;
They understand our production failures modes; how an error in a certain component will cause an error propagation across other areas. Production is a complex system;
They understand the risks involved in manual changes to production (adhoc scripts, changes to feature flags) and design follow a process to prevent mistakes;
They understand how important it is to identify root causes/contributing factors of incidents;
They understand exactly where are their worst tech debts, what is slowing the team down and causing more problems to customers. They push to prioritise the biggest pain points, because they know that they need that to grow responsibly;
They know which services they own, and they try to always improve on-call and customer experiences;
They understand that the only thing that looks like production is production; instrumentation (via tools like NR or exposing endpoints, improving logs, metrics, endpoints) is your only tool to debug production error;
They understand that CI/CD pipeline should protect you from feature bugs;
They take performance and reliability considerations (for the whole production) for every design, deployment and incident response;
The whole team shares the alarms/incidents role, as they understand the importance of having
They understand the expected availability, and
