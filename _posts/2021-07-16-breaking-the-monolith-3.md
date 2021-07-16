---
layout: post
title: Breaking The Monolith III
subtitle: Inter Service Communication
tags: [monolith]
image: /assets/img/btm_interservice.png
comments: true
---

One of the biggest decisions we need to make when breaking a monolith is: how will my services talk to each other? In a monolith, communication is synchronous by default; now that we have multiple services we need to decide what to use.

<div class='aside'>
This is post 3 of a series; I’d recommend reading at least the introduction of <a href="/2021/07/01/breaking-the-monolith-1.html">post 1</a> before continuing to explain the scenario.
</div>

As discussed in [post 2](/2021/07/09/breaking-the-monolith-2.html), our webhook service needs to expose a synchronous endpoint to enable the monolith to create and change webhook endpoint configuration (using a 2-phase commit). This means that our monolith needs to be able to send synchronous messages to the webhook service.

We also need async communication: our monolith needs to emit `employee_created` events which our webhook service can subscribe to so that it is eventually consistent. This allows the monolith to serve requests even if our webhook service is down.

We want to choose two technologies, one for sync and one for async communication, which can be used throughout the product. This will allow teams to share tooling and make integrating new services with each other more straightforward, and also spread learnings across the engineering teams. The technologies involved are very flexible, so until we come across a problem that cannot be solved by the ones we’ve already chosen, we want to stick with them.

# Synchronous Communication
The main choice here is between REST and RPC. RPC has different performance characteristics which may make it more appropriate for particular use cases, but make sure to consider the development experience including testing and debugging before committing to a particular technology. For us, it’s an easy choice to use REST. We wouldn’t benefit from the performance characteristics of RPC and we’re already familiar with the fantastic REST tooling from the OpenAPI project. Additionally, using RPC in a language that isn't strongly typed is not an experience I'd like to repeat. 

# Asynchronous Communication
The big name technologies for async communication are Kafka, AWS Kinesis, and Google Pub/Sub. They’re all very robust and while there are subtle functionality differences they will all solve most use cases. We choose Google Pub/Sub, because the rest of our architecture is built on Google Cloud Platform so we can plug into existing access management and infrastructure tooling.

<div class='aside'>
Terminology: Our monolith will publish <b>messages</b> to a <b>topic</b>. Our webhook service can then <b>subscribe</b> to that topic, to receive a stream of messages. 
</div>

Alongside Pub/Sub, to make our async communication infrastructure robust and resilient, we'll need some more capabilities:

## ❓ **A queryable record** of every event sent to a particular topic. 
This is very useful when trying to debug issues and is a prerequisite for being able to recover from outages (see below). We can set up a serverless function (using Google Dataflow) which also subscribes to our topic and pipes all the messages into a big-data store. We can then build a generic tool so that everyone using Pub/Sub can easily re-use our work, and engineers know where to find the record of events if needed.

## 📜  The ability to **replay history**
Imagine that our webhook service goes down and we need to restore it from a backup. We can run some code to pipe the data from our queryable record back onto the Pub/Sub topic since [x]. This will bring our webhook service back up-to-date. We don’t want to wait until something goes wrong to have to build this capability as it’ll likely be part of a wider incident response, and we will be in a hurry. Having this ready build, tested and documented is a big win.

## ❌  The ability to handle **unprocessable events** so that:
1. We are alerted *loudly* when an event cannot be processed (transient errors should be retried to avoid alert fatigue)
2. We are able to reprocess the event once the webhook service’s consumer code has been fixed so that the event is no longer unprocessable
3. We are able to 'abandon' the event (with manual intervention) once comfortable that the event cannot be processed (provided this is acceptable in the product context).

## 🔢  The ability to handle **misordered messages**.
Imagine we decided to use asynchronous communication to handle updates to the webhook configuration. We still want the ‘last wins’ behaviour for our updates which we’d expect from a database. This has some advantages (e.g. our replay history tool would now work for these updates, as well as just events), but clear disadvantages (the API can’t know if the update has been successful, and we now have to solve this pesky message ordering issue).

There are a number of strategies to handle misordered messages, one of which is to use sequence numbers. For this, we’d need the monolith which is serving the API request to store an incrementing sequence number against the customer’s account which changes whenever the webhook configuration changes. The monolith could then stamp a `last_sequence_number` on each event to ensure that the webhook service processes them in order. If the most recently processed `sequence_number` does not match the `last_sequence_number`, then the webhook service knows that it missed a message and can temporarily reject this message. Our service now needs to alert when it has temporarily rejected a message too many times, and an engineer needs to be able to resolve the issue (either by re-emitting the event or manually incrementing the expected `sequence_number`).

For us, this is complexity that we could do without (and isn’t adding much value) so we are going to avoid async messages where ordering is important and sidestep this concern.

# Interfaces & Schemas  

We should treat internal interfaces with the same care and attention as a publicly exposed API. To make it easy for other teams to use our service, we need to build some tooling to help us.

## 🧱 1. Build a Schema
The centre of our tooling is a schema, which is a static declaration of what we expect our interface to receive (and, for two-way communication, return). We are using JSON payloads, both for our sync REST APIs and for our async messages, so we choose to use [JSON schema](https://json-schema.org/). This allows us to use tooling such as [OpenAPI](https://swagger.io/docs/specification/about/) and [AsyncAPI](https://www.asyncapi.com/), which both use JSON schema to define their payload structure.

## ✅ 2. Validate Payloads
Our next step is to build some lightweight tooling to allow us to validate payloads against this schema. We want the schema to be accessible by multiple services, so we create a small library with a wrapper that allows services to validate a payload. Luckily, there are multiple libraries that will validate a payload against a JSON schema for us, so we just wrap that library (plus our schema) in a nice interface. It’s safest for us to validate the payload ‘on both sides’: i.e. both in the monolith and the webhook service. This means we can run tests on a single service while still being confident that the payloads are valid.

It’s important to validate payloads in production, not just in tests, so that you find edge cases quickly. It also allows the rest of your code to trust that the schema has been enforced, and avoids lots of ‘if this is blank then raise’ boilerplate. This is particularly important for async communication: as we’ve discussed, cleaning up bad async messages is really messy, so anything that gives us confidence that the messages are correct (even just structurally) is a good thing. This means that you can be confident that your schema is correct: it’s not just a point in time document but something that is validated in production every day.

## 📄 3. Document and Share
Now that we have a schema that is correct, we should generate some documentation so that other engineers can easily use our new interfaces. Again, there are a multitude of tools (e.g. JSON schema) that will generate a docsite from a schema, although this will only be as good as the descriptions and examples that you’ve written. Depending on the size of your team, you may want to include some narrative guides to accompany the raw payload structure documentation. We also need to host these docs somewhere that developers can find it: in our case we are using [Backstage](https://backstage.io/) as a service catalogue and it makes sense for the interface docs to live there.

Another useful tool to help our engineers be productive is simple client libraries. We can add more features to our schema library so that we create a ‘Client Library’, just like we would for our public API. There’s great tooling for auto-generating OpenAPI client libraries, so we can simply use that. It generates code snippets, which also goes into our backstage documentation. This means that engineers from other teams can self-serve to use the interfaces we provide, and have confidence in their implementations.

## 💔 4. Breaking Changes
It’s impossible to define an interface on day 1 which will last forever. We know this: that’s why we stopped doing waterfall software projects. In this situation, a breaking change is any change to our interface which breaks an assumption made by another system (see [How to Stop Breaking Other People’s Things](/2021/05/31/euruko.html) for a more detailed definition).

We want to know up front how we will handle breaking changes so that we can build both the tech and the processes to manage them. This comes back to one of our key costs of microservices: in a monolith you can deploy concurrently to multiple components as they belong to the same deployable unit. Once we split into services, everything has to be nicely backwards compatible to avoid downtime.

Breaking changes are a pain, and so of course the easiest path is to avoid them entirely. For example, we want to make sure we can add a new field to a payload without breaking everyone’s stuff. Our validation logic is nicely centralised, so we can simply ensure that the library ignores (and perhaps strips out) any unrecognised fields, rather than erroring. However, some changes will always be considered breaking (e.g. removing a field).

To support breaking changes, we need to add a version number to our schema. All events will be emitted with a `version` tag so that consumers know which schema should be applied. We also need our docs (and our schema library) to support multiple concurrent versions. We then need to define a process of ‘how do we make a breaking change’, both technically and organisationally. 

In HR-Land, we used to have a boolean flag `is_contractor` which we used to identify contractors (as opposed to full time employees). We’ve now learned that we need more granular detail and have decided to add a new enum called `employment_type` (which wasn’t a breaking change). However, we don’t want to leave the boolean flag lying around as we can’t reliably derive it from our `employment_type`, and so any consumers that are using it might make bad choices.

Firstly, we release a new version of the schema. This should flow through to our docs (so the latest docs don’t have the `is_contractor` flag) and our library (so we release a new version that includes both the old schema and the new one).

Then, we give other teams a heads up that the field is being removed on a particular date, and ask them to upgrade to the newest version of the library and adapt their code so it can handle payloads both with and without the `is_contractor` flag. 

On the pre-agreed date, our team starts sending messages based on the new schema (provided we’ve checked that none of the critical functionality will be impacted). This means that if a team hasn’t made the change in time, their service will start to error as the schema has changed. If these teams are consumers of async communication such as events, this may not be a problem: perhaps that service is deprecated or it’s an edge case which isn’t a priority for them.

There are clearly organisational challenges here as it requires co-ordinated work across multiple teams. That co-ordination isn’t cheap or easy, and it’s where you’ll find much of the hidden cost of multiple services.

# A Final Note on Idempotency
For the love of all that is holy, **make everything idempotent** or it will be very sad. When a request is idempotent, it means that if the caller makes the same request multiple times, the receiver will action the request exactly once, and return a consistent response. Without idempotency, many of our transactional guarantees fall into dust and you’ll have weird and hard-to-debug edge cases appearing all over the place. It also means things like being able to replay history just won’t work correctly as an instruction might be incorrectly actioned a second time. Some technologies are quite good at exactly once delivery, but only guarantee at least once delivery. We’ve chosen to explicitly check that our async consumers are idempotent by deliberately resending instructions and ensuring the results are as expected.
