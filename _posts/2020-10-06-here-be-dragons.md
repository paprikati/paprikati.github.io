---
layout: post
title: Here Be Dragons
subtitle: Dealing with code you don't understand
tags: [tech-debt, scale]
comments: true
---

As you grow a codebase, eventually (or perhaps quite quickly) certain parts become 'scary'.
"Here Be Dragons" is an actual quote from a codebase I've worked on. Let's talk about how to
manage this kind of code.

## Why does stuff get scary?

### If I break it, it's really bad

All code may be equal, but some is more equal than others.
The bit of code that means we take the wrong amount of money from a customer is more scary than the bit of code
which internationalises the API error messages because the impact of it breaking is higher.
This isn't really fixable - some stuff is important - but means you'll have to think more carefully
about testing and release strategies in these areas.

### I don't know what it's actually doing

Some code is hard to read. Maybe it:

* is doing something really complicated
* was written in a rush
* is in an unfamiliar language
* has been vastly adapted from its intended use-case
* was written by someone with an *unusual* style

However it happened, a developer now reading the code for the first time (or maybe the tenth) struggles to understand
the flow of data through the system. This often also comes alongside limited test coverage or documentation.
This makes it scary because making changes safely is hard - it's difficult to be confident that what you are
changing will have the desired effect.

### I don't know why it's doing something

Sometimes I encounter code where I don't know *why* it's there. I can see *what* it is achieving (that's the readability point
above) but there's nothing in the comments / commit history / institutional memory which tells me why we've decided to do that.
This usually results from fixing bugs, and is the primary reason why commit messages are so important to maintaining your code base.
It also means I can't distinguish between fixes for bugs that are implementation specific (and therefore I can ignore if I change the
implementation) and logic which is required no matter the approach. Some of this can be caught by tests, but often not all of it.
Additionally, tests can be hard to re-use if you are making fundamental changes about a system.

{: .box-note}
**Can we just, not screw it up in the first place?**<br>
Of course this is the ideal solution. If we keep these considerations top-of-mind then our code is less likely to become scary.
The sad reality is that often parts of our code bases, despite our best efforts, do get into these scary states and so we need to learn
to manage them.

## So, What Next?

There is always the 'Do Nothing' option. And it may work for you for a long time: if it ain't broke, don't fix it, and all.
However, at some point this will no longer be the correct option. Leaving scary code unchecked causes all
sorts of long-term issues: features get abandoned because 'we can't change the scary code' and incidents happen when the code doesn't
behave how you expect or account for some specific situation. There are a few things that might tip you over the edge here:
often it's an incident, sometimes it's a feature request, sometimes it's an engineer with a personal grudge or if you're doing
really well it might have just bubbled to the top of your tech investment to-do list. 

The rest of this is a step-by-step guide of how to 'deal' with scary code - which is all based on the premise that you are trying
to refactor a system without significantly changing it's behaviour. Not everything will be relevant for everyone, but I hope
there'll be some useful advice for a wide range of use cases.

### Step 1: Identify The Scary Stuff

Having regular discussions about which bits of code are most 'scary' should be a normal part of managing your code base.
Most developers make these judgements quite naturally - usually independently from one another - when estimating the
complexity of a project or ticket. However, talking about it and aligning can be really productive by:

* sharing knowledge about where scary code might be hiding
* identifying who has the most context on a given scary thing
* finding the scary code that is most likely to bite you next (and prioritising fixing it)

### Step 2: Small, Incremental Refactors

This step is primarily targeting the readability problem. You probably also want to add some tests here (if
the coverage is poor) to help give confidence that you aren't changing any behaviour. They can also act as documentation
for the expected behaviour of the system. The great part about this step is that you can do it incrementally in
between bigger pieces of work - you'd be surprised how quickly you can increase the readability of code with some
sensible variable name choices / re-organisation of classes. This process will also help you understand more about
the scary code, and it's likely you will find more of the 'I don't know why this is here' code. We'll deal with that
in step 3, but it's worth adding comments as you go, for example:

```ruby
# TODO: We don't know what this is doing, we suspect it's something
# to do with batching
```

You may find that this process moves the code out of the `scary` bucket, in which case congratulations, you can go back
to building features.

### Step 3: Investigate

This is mostly to understand why the code is behaving in a certain way. Your first port of call here should be the code itself
(and comments), followed by a `git blame` to find any commit messages or PRs that might help you out. You also probably want
to ask the longest-tenure engineers if they remember anything, and also dig through any documents they might have written at
the time. Discoverability of documentation is hard, and most companies aren't very good at it, so have a think about what will
make this easier next time round. This is often the bit which gets harder over time, particularly the 'ask the engineer' part,
which is one of the reasons leaving scary code alone is risky.

### Step 4: Observe & Compare

Assuming all of that fails, we then get to the 'observe and compare' part. Try to isolate bits of code which don't make sense,
keeping them as small as possible. Then add a flag to disable that logic / use something different. Then write something 
(probably a low-priority asynchronous job that's picked up by a worker) that runs the code without that logic, and compares
'what we actually did' vs. 'what we would have done without the unexplained logic'. This comparison obviously depends on the
type of change - you might be comparing some data output or performance metrics. You can then log any times where the results
are 'different' (whatever that may mean), and then manually inspect them and see if you can understand why the code is there.
If the results are not meaningfully different for a large enough sample size (this one's up to you), then probably change the
logic in the existing code. 🤷.

{: .box-note}
For performance reasons you may want to use sampling so you don't double-process *everything*, e.g. enqueue a comparison worker
for 1% of requests.

### Step 5: Refactor, Rebuild or Walk Away

Once you have a clear understanding of what your code does and why, it's lost most of its scariness. The main thing left
is probably importance - or 'cost of failure'. There's a fork in the road here: do you want to refactor the code (i.e.
return to step 2) or do you want to rebuild something new? Or have you solved / abandoned the original problem?
The trade-offs there are out of scope for this post, but either way the first 4 steps should put you in a much better
position to make that decision.
From here on in, the approach to this is the same as any risky change - thinking about silent releases, cutover hygiene and observability.
That's a topic for another day.
