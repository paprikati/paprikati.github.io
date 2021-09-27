---
layout: post
title: Complexity and Risk
subtitle: Managing complexity and risk over time
tags: [complexity risk]
image: /assets/img/complexity.png
comments: true
---

In software, complexity comes from multiple components interacting with each other in different ways. We can imagine that all code is constructed from boxes which take some data as an input, do something, and provide data as an output. The more of these boxes we have, and the more logic we have connecting them, the more complex our system becomes.

It’s difficult to provide value without adding complexity. Imagine that we started a company which provided an API which would accept an English string and a target language and return a translation. There’s hidden complexity in this product: we need to have infrastructure to be able to serve these requests, perhaps some validation to make sure they’ve provided a valid string with characters that we recognise. We might need some authentication so that only people who have paid for the service can have access. Quickly, our simple product becomes something much more complex.

The aim of a project is often to add or enhance a feature. This usually means adding complexity to the system. As you scale your engineering team, and that team has time to build more features, the complexity of the product will continue to grow.

# Complexity = Risk
As your systems become more complex, the risk that something will go wrong increases.

Let’s get back to our translation start-up. Let’s say we wanted to add a feature so our API accepted a `source_language` parameter as well as `target_language`. So we diligently add the feature so that you can now specify the `source_language`. Ignoring the actual translation logic (let’s assume we are using google translate in the background), we’ve added many different ways our code can fail:
What if someone provides an invalid `source_language`?
What if the `source_language` is the same as the `target_language`?
What if the text isn’t in the `source_language`?

Similarly, imagine our translation start-up was getting lots of traffic, so we decided we’d like to add a load balancer to support multiple back-ends to serve all our requests. This is now a whole new service that could have an outage; even if we can keep the back-ends always alive and ready to serve requests, that won’t matter if the load balancer is dead.

As the systems become more complex, it becomes more difficult for engineers to reason about all the edge cases and failure modes of the various components. That means engineers make more mistakes, and we have more problems. These problems can take many forms, from service degradations or downtime through to business logic errors (charging someone the wrong amount of money) or security issues (allowing someone to see something they shouldn’t be able to see).

It also becomes more difficult to fix things when they do break. As our systems grow more complex, we experience outages where multiple components fail at the same time, in a new and unexpected way, which can be notoriously hard to debug. Imagine our translation start-up had decided to add both a load balancing service and also an authentication service. Someone has called our support team saying that they are receiving 503 Gateway Timeout errors from our API. We now have to look at each service in turn to identify where the problem even is, before we can start thinking about resolving it.

# Not all complexity is equal

There are certain types of complexity that we should be particularly careful about.

## 🕸️ Distributed Complexity
Back to our start-up. We’ve decided that we want to introduce a two-tier pricing system. People on the cheaper tier (tier 1) can only translate English strings, but those on the more expensive tier (tier 2) can specify the `source_language`. We also want to allow tier 2 customers to access a new thesaurus endpoint which would return multiple alternatives for the provided word.

This doesn't sound too difficult for us to implement. In the most naive approach, we would (in pseudo-code):
```
// In the translation endpoint
if (request.customer_tier == 1 && source_language != 'en'){
    return 'Forbidden'
}

// In the thesaurus endpoint
if (request.customer_tier == 1) {
    return 'Forbidden'
}
```
We now have two different places where the code needs to understand (and check) that we have multiple pricing tiers, resulting in **distributed** complexity.

The best way to manage this kind of risk is to isolate complexity into its own component and wrap it in a single clean interface. We could have a bit of code shared across all endpoints (often called `middleware`) whose job it is to check the customer’s tier and then apply a set of rules about whether they can take the action or not. This would mean that our complexity is no longer distributed: there is a single bit of our code which knows about this business logic, and no-one else needs to. Our [permissions project](TODO) at GoCardless is a great case study of this kind of project.

## ✖️ Multiplicative Complexity
Our start-up is using a third party to provide the thesaurus endpoint, but they aren’t licensed to operate in Germany. That means that even if a customer is on tier 2, we still can’t allow them to use the thesaurus endpoint if they’re using a German IP address.

Again, this doesn’t sound too difficult. We can get our load balancer to tell us which country any given request comes from. Then we just need to edit our logic above:
```
// In the thesaurus endpoint
if (request.customer_tier == 1 || request.country = ‘Germany’) {
    return 'Forbidden'
}

```
When we combine that with the two-tier pricing system, we now have 4 different versions of the endpoint that we are supporting: German tier 1, German tier 2, Other tier 1, Other tier 2. This is **multiplicative**, and as anyone who’s read the ‘rice and ches knows: when you start multiplying numbers together they get real big, real fast. It’s easy to incrementally add more and more branches to endpoints like this, until they become really difficult to reason about and debug.

Isolating complexity can also help us here: if our code to manage the customer tiers was nicely separated from the country-specific restrictions, this multiplication becomes less problematic. If none of the rest of the code is worrying about which tier the customer is on, there are fewer scenarios to test and reason about when implementing the German restriction.

## 🕶️ Opaque Complexity

Our start-up has so much traffic, that we’ve decided we can no longer scale our MySQL server to handle our data so we’re moving to Cassandra. We’ve not used Cassandra, but it’s a popular product and we’re not doing anything too unusual, so we’re confident we can manage it.

As experienced by [Monzo in 2019](https://monzo.com/blog/2019/09/08/why-monzo-wasnt-working-on-july-29th), Cassandra won’t always behave in the way that you expect. The thing that makes this so dangerous is that the system is incredibly complicated that it becomes **opaque**: unless you are a Cassandra expert it is very difficult to get debugging information about why your Cassandra database isn’t behaving as expected. Using 3rd party technologies is often the right decision (as against building things yourself), but for highly complex and critical systems it carries significant risk.

One solution to this would be to build up more Cassandra expertise (either in-house or by hiring). An alternative approach would be to procure a managed version of the software (there are many to choose from), thus hopefully entrusting your system to a more experienced group of engineers for whom the software is less opaque. The other answer is to use more ‘out-of-the-box’ style technologies, which require less careful configuration and tuning, at least for as long as you can.


# Managing Risk

As a product grows, the complexity (and therefore the risk) grows alongside. We can think about this as taking out debt, and using our talented engineers to pay the interest.
We pay the interest in two ways:
1. It takes us longer to build and ship new features because we need to gain confidence that our systems work as expected.
2. Things go wrong in production, and it takes us time and money to resolve the issue and compensate the affected customers.

Over time, paying off this interest can slow your product organisation almost to a complete halt. It’s important for engineers (particularly those in leadership roles) to be able to communicate clearly about the risks that the product is carrying, and identify potential projects to help reduce those risks (reducing the interest we need to pay).

We’ve already discussed some of them: isolating complexity and building expertise are both great strategies for reducing complexity. It’s also important to constantly trim unused or unstrategic functionality, which is ubiquitous across most start-ups. The functionality doesn’t appear to have a high maintenance cost, so sits untouched in the code base and slowly rots. If you have 10,000 customers and only 12 of them use the dictionary endpoint, unless it is deeply strategic, it’s a good candidate to be removed. Alternatively, it might be a configuration option that is rarely used or a workaround for a bug that’s since been fixed. The engineering effort required here is usually pretty limited as it’s often simply deleting code, but there may be additional effort required to communicate the change to customers. Having a well-trodden path that makes removing functionality easy is an important part of keeping complexity under control. This is sometimes described as ‘weeding’ - removing the functionality you don’t want to give space for new functionality to grow up in its place.

# Testing, Testing, 1, 2

Projects to make changes like isolating complexity or removing unnecessary features can be quite scary. That fear can result in teams leaving lots of code untouched for years on end, for fear they will break someone’s integration. Tests are our best weapon in this particular battle. If we want to be able to safely and easily refactor and isolate complexity, having robust integration / end-to-end tests which demonstrate that the behaviour of the system remains unchanged is critical.

# Wrap-Up

We cannot build useful products without complexity, so avoiding complexity all together is clearly not a useful strategy. But we should view all new features as a trade-off between the value they provide and the risk they carry. It's often worth questioning whether the desired customer outcome can be achieved in a less complicated way: perhaps by changing some existing business logic or changing the implementation details. We can use the strategies outlined above to manage our risk, allowing us to deliver value while keeping our systems robust and resilient. Happy coding!
