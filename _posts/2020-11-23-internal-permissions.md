---
layout: post
title: Who can do what?
subtitle: Managing permission rules
tags: [tech-debt, scale]
comments: true
---

Earlier this year, we overhauled our permissions system at GoCardless. It's given us a platform to build
exciting tooling, as well as increasing our confidence and visibility of the system. This post shares some
of the thought processes and lessons learned.

The aim of the project was to pull together all the code we had that answered the question 'Can I Do This Thing'.
For us, the main focus was our API - we have a vast number of endpoints with subtle rules applied governing who
can do what. We had three objectives:

1. **Visibility**: anyone can easily see who is able to use a particular endpoint.
2. **Confidence**: we can be confident in the rules we are applying, and can change those rules.
3. **Enablement**: build a platform to support future tooling.

## Rules

Any permissions system is built out of rules. A rule is a piece of logic such as 'users with read-write access
can do [x]' or 'accounts on the basic package cannot do [y]'. In my experience, these rules naturally grow
more complex over time, and a periodic review can be very useful.

## Permission vs. Restrictions

You can frame a 'who can do what' rule in two ways: either via **permissions** (rules that permit access) or
**restrictions** (rules that prevent access). Each model naturally enables different structures.
**Permissions** works well when most of your rules are applied using a `UNION` - i.e. if you fulfill criteria (a)
**or** criteria (b) then you can do the thing. **Restrictions** works better when you are using `INTERSECTION` - i.e.
id you fulfill criteria (a) **and** criteria (b) then you can do the thing.

You may need to use both in your system, but try to avoid it if you can as it makes your rules harder to define
and follow. I'd recommend using a hierarchy if you need - so either having a top level **permissions** model where
some of your rules include `INTERSECTIONS` inside, or a top level **restrictions** model where some of your rules
include `UNION` inside. 

## Scoping the project

As part of scoping, we first had to catalogue all the rules we were applying in our API (which was a non-trivial
exercise, unfortunately). We chose to pursue a **restrictions** model, largely because that's how the existing
system was structured, but also because it better suited our use case where the majority of our rules were
combined using `INTERSECTIONS`. To be honest, I suspect this would be true for most public APIs.

Our next major challenge was to decide how we wanted to declare and manage the rules. We would ideally have liked
to encode all our restrictions logic into a token, which 'knows' what it can and cannot do. For various reasons
that wasn't possible for us right now, so all our thinking revolves around the idea that (for now) our rules are
applied by the endpoints themselves, at the point that the API request is made.

## Rule Catalogue

We quickly agreed that we wanted to have a catalogue of rules which could be applied to different endpoints, and
were well documented. The same code would run on all of our API requests, and apply the rules so we could be 
confident that they were consistent and correct. It also made our testing approach simpler as there was a single
component that could be battle tested rather than the logic spread across our code base. Additionally, because all
the rules were applied using an `INTERSECTION`, we could unit test each rule individually and then test
a few combinations, as the combination logic was trivial (apply all the rules, if you pass them all then continue).
Any complicated logic was contained within a particular rule, and then unit tested carefully.

## Which rules does each endpoint apply?

The next step was to understand how rules from this catalogue were applied to particular endpoints. We came up with
two contrasting approaches, both of which were in use in the API when we started.

### Option 1: Centrally Defined Restrictions

In this design you have a single central location which details which endpoints are included in a particular
rule, as well as a centrally defined logic for that rule. For example, you might have a single file which has 
a list of read-write endpoints which read-only users cannot access.

The key advantage of this model is that it's easy to answer the question 'what can a user with attribute [x] do'.
For example, what endpoints are unlocked by upgrading an account from basic to pro. This visibility can be useful
internally too - what extra permissions does a support team member get if I give them 'tier 2' access? 

There are options of how you store these restrictions: you could store them in a database rather than in code,
although we abandoned that idea due to the challenges of maintaining the rules across multiple environments.

### Option 2: Endpoint Owned Restrictions

Instead of centrally defining the restrictions, you can instead define them alongside the endpoint definition.
Each endpoint then 'owns' its own restrictions, and they can be easily reviewed next to the endpoint's code.

The main advantage here is that it's easy to answer the question 'who can access endpoint [x]'. For example,
what restrictions do we apply to someone wanting to create a new customer. It makes the rules very discoverable
(as they're in the endpoint definition itself) and makes the 'I forgot to restrict my route' bug less likely as
they restrictions are clearly part of the code review.

## What we did

We wanted to have our cake and eat it: there were proponents of both options with strong arguments, and for once,
we found a way.

We chose option 2 but added a script (enforced by our CI pipeline) that generates two JSON files from the endpoint
definitions and pulls all the applied rules into a single location. One file is `restrictions_by_route` which
answers the 'who can do [x]' question. The second file is `routes_by_restriction` which answers the 'what can
someone with [x] flag do' question.

## Bonus

This work enabled us to expose an endpoint that answered the question 'what endpoints can I access?'.

*N.b.: this endpoint already existed, but it only included about 30% of our rules*

This is a really powerful feature to power your UI - it allowed us to strip out duplicated logic in our
front end which was being used to conditionally render certain buttons, and rely completely on the API as our
single source of truth for who can do what. There are a few other ways of achieving this (we discussed using `OPTIONS`
calls everywhere instead) but this was a much simpler solution for us to implement.

## Double Bonus

The best bit about the new endpoint was that we could add a feature we'd wanted for ages: 'what endpoints could
I use if I wasn't a GoCardless staff user'. Our API has a number of features that are restricted to just GoCardless
staff. Some of these features are exposed in the dashboard, and to a new joiner it's impossible to tell what they
can do 'because they're a GC user' and what an external user would see. This makes onboarding really tough as our
support teams have to learn all the different rules about what features are available to external and internal users.
We now use this endpoint to conditionally render a bright pink box around any content which is only visible 'because 
you are a staff user', which has really helped our internal users.

<img src="/assets/img/pink-buttons.png" width="300px" >

My favourite part about this is that the code to achieve it in the UI is really simple:

```jsx
const UnlessUserIsRestricted = ({
  resource,
  action,
  isUserRestricted, // injected from redux
  wouldUserBeRestrictedIfNotGCAdmin, // injected from redux
  children,
}) => {
  if (isUserRestricted(endpoint)) {
    return;
  }

  if (wouldUserBeRestrictedIfNotGCAdmin(endpoint)) {
    return <div className="u-admin-only">{children}</div>; // this adds the pink box
  }

  return children;
};
```

```jsx
return (
  <UnlessUserIsRestricted resource="customers" action="create">
    <Button>Create Customer</Button>
  </UnlessUserIsRestricted>
)
```

## So How Did We Do?

1. **Visibility**: our JSONs provide a single source of truth for 'who can do what'.
2. **Confidence**: we have 700 lines of tests unit testing each of our rules, and we can easily change which rules are applied to each endpoint.
3. **Enablement**: we've enabled the bonus projects to reduce complexity and duplication in our UI, and help out our internal users along the way.

This might be the thing I'm most proud of that I've done at GC so far. I hope you enjoyed the read!
