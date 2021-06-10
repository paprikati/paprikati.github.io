---
layout: post
title: Reflections from GoCardless
subtitle: A few nuggets I've learned along the way
tags: [learning, onboarding]
image: /assets/img/gocardless.jpg
comments: true
---

This is the first time in my career that I’ve left a company I really liked, which was quite a strange experience. I wanted to share some of the things I’ve learned along the way. I hope to turn many of these into full blog posts, so consider this the bitesize version.

### Context vs Skill
When you join a new company, you need two things to progress - context and skills.
Context is ‘how do the systems work in this company / industry, and why’.
Skill is the transferable stuff: problem solving, design and architecture, communication etc.
I’ve found separating these concepts very useful, both for combating imposter syndrome
(that mistake was because I was missing *context* not *skill*) and also for building effective teams
(if you don’t have enough context or skill in a team, you’ll struggle to succeed).
It’s also a good way of prioritising your own work: in the first few months you’ll be focussing on context,
while probably picking up some skill on the way.
You can then pivot towards being more skill focussed (while of course still picking up more context),
at which point you can be considered ‘ramped’. 

### Find great people, and stick to them like glue
The best way to learn (for me anyway) is from other people who have context or skill that you don’t.
Strive to never be the smartest person in the room - find the people with skills you admire and try to
find opportunities to collaborate and learn from them.
Also ask them for frequent feedback - and try to be really open even if you think they’re being unfair.
Although it stings, there’s probably some truth in what they are saying that it’s worth you reflecting on.

### Learn from other people’s mistakes
Engineers learn a lot from making mistakes, which means your progression is rate limited by the speed at which you can make mistakes.
You can accelerate this process by talking to other great people, and learning from their mistakes.
Everyone loves telling war stories, so take someone out for a coffee and ask for their ‘most memorable mistakes’.
You can also use other sources such as post mortems (internal and external), and podcasts (e.g. the Downtime Project).

### Incidents are a great way of learning
Incidents (i.e. when something goes wrong) are a great opportunity to gather both context and skill.
If it’s possible to get involved, either in real time or after the fact, then do.
At first you’ll just be an observer (write down questions so you can ask someone after the immediate danger has passed).
Then as you gather context, you’ll progress to contributing:
initially in small ways (e.g. maybe this dashboard will help) all the way up to running your own incident response.
You’ll get tonnes of context, get to know people from across the org, and learn how to build resilient and reliable systems.
It’s a key differentiator between ‘good’ and ‘great’ engineers.

### Volunteer for things to make your own opportunities
It’s very rare that an organisation will push back if you offer to do something useful.
Of course there are some cases where this is hard (e.g. cultural resistance to a particular change,
or if you’re not delivering on your core role) but many people will have the leeway to try to push things they care about.
Use this as a lever to help you gather context or skill that you don’t have, and collaborate with people you want to learn from.

### Harness the power of new joiners
New joiners are awesome because they haven’t learned to ‘accept’ the things that are broken or bad about your systems and processes.
They’ve also probably come from a place that did something better than you.
Use this knowledge and energy to your advantage - particularly for a fast-growing company it’s a huge potential source of good things.

### Complexity = Risk
I’ll definitely write a whole blog about this one day. TLDR: complexity => risk => incidents => sad.
Complexity can be good if it helps your product be valuable; that doesn’t make it not risky, it just makes the risk worthwhile.
Fighting complexity is all about looking at the trade offs of the complexity you’re carrying vs. the product value it delivers.
This work often takes the form of removing tech debt or legacy features which can be unglamorous but is also really valuable.

### Your data is important
If you can’t archive your data to a place where the sun doesn’t shine (which at GC, for various product reasons, we can’t),
then it matters that it stays in a good state.
We’ve been tripped up a few times due to ‘old, bad data’ that no-one ever cleared up.
Whenever we leave data around like this, we are forcing future developers to
(a) know this weird thing happened (they won’t) and
(b) know how to interpret this weird thing (they don’t).
Cleaning up data is often a faff, but it’ll pay off in the long term.
