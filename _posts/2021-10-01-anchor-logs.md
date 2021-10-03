---
layout: post
title: ⚓ Anchor Logs
subtitle: Supercharge debugging with a handful of well known log lines.
tags: [observability logging]
image: /assets/img/anchor.jpg
comments: true
---

An Anchor Log is a carefully chosen wide logline which becomes the centre of your debugging journey, and makes it easier to navigate unfamiliar parts of your ecosystem. This blog post walks through my journey towards identifying **Anchor Logs**, and shows you how to get value from them on your own projects. 

# 1. Basic Logging

Logs are a really useful debugging tool. As the natural descendant of `console.log` debugging, my first web app printed loglines that looked something like:

```
11:37:29.827 [web] INFO initialising app
11:38:21.342 [web] INFO listening on Port: 3000
11:38:23.917 [web] INFO received API request /api/auth/login
11:38:24.012 [web] INFO returned 200 for request /api/auth/login
```

While definitely better than having no logging, this isn't exactly 'state of the art'. 

# 2. Structured Logging

As soon as you start wanting to do any kind of aggregate analysis, or searching and filtering, the unstructured logs become difficult to manage. Most people now use structured logs, usually JSON, that look something like:

```json
{
   "@timestamp":"2021-10-01T11:37:29.827Z",
   "event": "api_request.received",
   "ip": "8.8.8.8",
   "level":"INFO",
   "msg":"received API request /api/auth/login",
   "path":"/api/auth/login",
   "request_id": "abc-123",
   "user_agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/77.0.3865.90 Safari/537.36"
}
```

This is great! We can now start to analyse these logs using computers as they're more easily parseable. We can throw these into (for example) a big ElasticSearch cluster and do searching, filtering and aggregation. I'd highly recommend enforcing that all loglines have:
1. a human readable message (`msg` in our example) to help with narrative
2. a machine readable key (in this case `event`) which can easily be filtered and also searched in the codebase.

As you work on a codebase for a while, and debug lots of different issues, you'll find that there are a handful of loglines you use more than the rest. Often there's a logline with the duration and status of an HTTP request, which is really useful when looking for patterns across your API. There's also likely to be a logline fired when you enqueue, begin or complete a job, which is great when looking at queue latencies or trying to work out which jobs ran on a particular day. In my experience, these emerge naturally as a 'happy accident', and knowledge of them lives solely in the minds of the engineers doing the debugging. But we can do better.

# Introducing: Anchor Logs

Let's start with a premise: every unit of work should have a single Anchor Log, which has as much relevant information as possible. A unit of work is likely to be either a request from the outside world (HTTP / RPC request) or a job (triggered from a queue, an event or a scheduler like a cron).

You'll want all the information, including how long the unit of work took, so you probably want to emit this right at the end of the request/job.

Our API logline might look like:

```json
{
   "@timestamp":"2021-10-01T11:37:29.827Z",
   "duration": 0.234,
   "endpoint_method": "login",
   "endpoint_service": "auth",
   "event": "anchor.api_request",
   "http_method": "POST",
   "ip": "8.8.8.8",
   "level":"INFO",
   "msg":"⚓ processed API request /api/auth/login",
   "organisation_id": "O123",
   "path":"/api/auth/login",
   "request_id": "abc-123",
   "status": 200,
   "user_agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/77.0.3865.90 Safari/537.36",
   "user_id": "U123"
}
```

And our job logline might look like:

```json
{
   "@timestamp":"2021-10-01T11:37:29.827Z",
   "duration": 0.234,
   "enqueued_at": "2021-10-01T11:33:22.706Z",
   "event": "anchor.job_worked",
   "job_args": "[\"U123\", true]",
   "job_class": "Jobs::DoSomeWork",
   "job_id": "J123",
   "level":"INFO",
   "organisation_id": "O123",
   "queue": "default",
   "result": "success",
   "started_at": "2021-10-01T11:35:18.162Z"
}
```

But, I hear you ask, why have you written a whole blog post about these? And given them a stupid name?

Well, because they're *super useful*, and knowing you've got them means you can double down and get even more value from them.

## 🐛 Debugging

When you're debugging an issue, it's really useful to have an entrypoint into 'what happened in this time period' or 'what did this user do'. Anchor logs are a great answer here: if you search `event:anchor.*` in your log explorer of choice, you can filter to a time period and perhaps an organisation/user and see at a glance all the key things that happened.

## 🛳 Onboarding

When you become familiar with a codebase, you start to learn how to find the key log lines, and roughly what shape they are (i.e. should I be filtering on `status:500`, or `http_status:500`, or `http_code:500`). If you have a really small number of anchor logs, and they're documented well, it's much faster for a new joiner to get up to speed and become a debugging wizard. Navigating your logs stops being a dark art that has only been mastered by an elite few. If everyone is debugging in the same way, it becomes much easier to communicate during the heat of an incident and allows you to invest in tooling for these specific code paths (e.g. pre-defined links to certain useful queries and visualisations).

## 📊 Aggregation & Analysis

While metrics are the ideal tool for aggregation and analysis, often I've ended up analysing logs as we haven't had the metrics in place for the time period we're considering. This is particularly common in incidents where:

(a) something very unexpected has happened and

(b) it's probably happened recently so well within your log retention window.

When analysing logs, you're generally faced with a much more limited query capability that doesn't allow you to easily join information from different sources. Having anchor logs that are information rich allows you to easily answer questions like 'are all the 422s related to one organisation' or 'were all the failed jobs enqueued after a particular timestamp (perhaps a deploy?)'.

# Putting it into practice
Making anchor logs work shouldn't be too big an up front investment, and it's something you can iterate and expand over time. The likelihood is that you already have something similar that you can use as a baseline and expand upon.

## ℹ️ Get all the information
To make it easier to find the loglines you want, either looking for a specific user journey or for aggregation, you need to have lots of information on each log line. This is likely to include customer identifiers like `organisation_id` and `user_id` (in our example above). These are likely not to be immediately available wherever you log the anchor log (which, to get the duration, has to be right at the outer layer of your handler) so you'll need to write some kind of abstraction to stash that information when you get it (e.g. in your authentication middleware) and pull it out when you log. You also want all the relevant timestamps so you can filter by those (e.g. `enqueued_at`) - generally speaking the cost of adding fields should be low, as long as they're named sensibly. Try to keep consistent conventions across the different anchor logs: use the same key for your `duration` field across all of them, so no-one has to remember which one is which. (P.S. also use the same unit!) 

You can read more about the benefits of wide events in this [twitter thread](https://twitter.com/mipsytipsy/status/1009541219729281024?lang=en) from Charity Majors, CTO of [Honeycomb](https://www.honeycomb.io/).

## 📄 Documentation
Write up documentation of what anchor logs are, and what your ones look like. If this can sit alongside the code that's emitting the logs and keep the docs in line, that's an added bonus. This then becomes the perfect onboarding material for new joiners struggling to penetrate the large pile of logs they are faced with when they get given their Kibana login (or equivalent). You can also build useful queries and visualisations for people to piggy-back on, making debugging even easier and faster.

## 🕵 Track down those missing logs
Your logging systems are unlikely to have 100% retention guarantees (often there'll be 'best effort' components), but you want to make sure you're getting the vast majority of your logs. Wondering 'did we drop the log' is added complexity which no-one needs when debugging an issue. I've seen people claim 'logs are bit lossy' as an explanation before really investigating any other alternatives. If you encounter more than one missing log in a debugging context, you're really unlucky. Take the time to investigate other possibilities - there's often a component shouting at you if you look hard enough. A few possible failure modes I've encountered are:
* Application logic where logs are not issued in certain cases (e.g. only logging for jobs that explicitly call a 'logging' function)
* Situations where logs are never issued (e.g. if the request is timed out by the infrastructure layer)
* Exceeding a single logline size limit, so something is truncating and mangling logs to make them unfindable, or dropping them all together
* Exceeding a buffer size, so something is dropping excess logs
* Falling foul of a schema somewhere - did someone send a duration as an int and now your log ingester discards everything where duration is a float?
* Pods being killed before they have time to get their logs somewhere persistent

If you have a low tolerance for missing logs, you will likely find and fix these issues and you can continue to rely on the presence of your anchor logs in almost all cases.

# When should I use anchor logs?
Any unit of work - an incoming request or async work (i.e. a job) should definitely have an anchor log. There are a few other places that you might want to have an anchor log, depending on your architecture. Having an anchor log when emitting an event can be really useful, or perhaps having one for every third party API call. The key here is keeping the set as small as you can, while providing maximum value. If there are too many, engineers can't keep track of them and they cease to be useful. It's also important to keep them consistent - if the incoming HTTP request logs look pretty similar to the outbound ones, that's less information an engineer has to remember to be able to easily navigate the logs. You'll also need to think about your logging infrastructure and storage costs: issuing and storing a log isn't free, and so there may be situations where you need to apply sampling techniques or lean more heavily on metrics.

# Is this a log, or a trace?
As well as logs, many observability setups now use traces and spans. It's possible to implement anchor logs as traces: you have one trace per unit of work, and they contain lots of useful metadata about the work being done. You can then have a bunch of spans inside the trace which are linked using a `trace_id` and your observability platform of choice will render it all beautifully together. Whether you use logs or traces is a matter of preference, likely informed by your specific observability setup. Different tools allow you to explore, visualise and aggregate logs and traces differently - you may need a trace, a log, or both to get all the value that you want.
