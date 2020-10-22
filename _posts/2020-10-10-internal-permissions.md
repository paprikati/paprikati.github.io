---
layout: post
title: Who can do what?
subtitle: Managing permission rules
tags: [tech-debt, scale]
comments: true
---

Who can do what?


Opt 1. centrally defined restrictions

1.a) db
Pros - great for configurability (e.g. if you want to let each customer configure their rules, its the only way)
Cons - hard to keep in sync between multiple envs. Hard to give a good UI to give good discoverability.

1.b) in code
Pros
Cons


Opt 2. action owned restrictions



What we did

our starting point wasn't great. big mix of crap

we went for option 2, with a script that generates a declarative set that can be used for all the above stuff.


Bonus

we also enabled a 'what can I do' endpoint, which allowed us to build features including (if I wasn't a staff member) and 
hide irrelevant UI elements so we don't need to work to keep the front end and back end in sync.
