---
layout: post
title: "Put Your Hand Up"
subtitle: Keeping a small-company mindset as you grow
tags: [learning, culture]
image: /assets/img/hand_up.jpg
comments: true
---

There’s always tasks in a business that don’t have a clear owner. If you can change the culture from "it's not my job" to "I'll handle that", there's much less friction and the walls between teams start to come down. The solution? Teach everyone to 'put their hand up'. And then reward them for it.

--

In a team meeting, my product owner asked ‘how many partners do we have that only use read-only features?’ Everyone looked away from the camera, hoping that someone else would volunteer. The product owner, reacting to the silence, started to explain why it was so important to have this information. Eventually, I volunteered.

### Principle of least discomfort
The silence caused social discomfort among the team. People have higher or lower tolerances for this, and the person with the lowest tolerance will usually end up volunteering. Social capital contributes to a person’s tolerance: long-standing or senior members of the team are more likely to be comfortable with the silence than a newer or more junior team member. There are diversity effects: women and minorities generally have less social capital, so volunteer more readily.

The result is that these ad-hoc tasks are not distributed evenly across the team, and instead fall on one or two individuals. This can mean that these individuals struggle to meet other commitments, or get frustrated by always having to do the tasks that nobody else wants. This experience is also likely to make the product owner think twice before making a similar request in the future as they also experience the discomfort.

To avoid this scenario, we need to encourage our team members to volunteer for these kinds of ad-hoc tasks – or ‘put their hand up'.

### Growing pains
Once a product organisation grows above a certain size, it needs to divide itself into teams. While this is a blatantly sensible choice, one negative consequence is that work which doesn’t clearly sit in one team is often neglected.

#### 🧑‍🤝‍🧑 Identity
Individuals will have stronger personal relationships within their team than with other teams. This means that pushing undesirable work onto another team is more attractive than it was when everyone considered themselves part of the same team.

#### 🥛 Capacity
If a team is under-resourced with tight deadlines, then the only possible response is to reject all work that isn’t directly contributing to the immediate project. If they view themselves as the busiest or the most important team, this can increase tensions even further.

#### 🧮 Accountability
If a team is measured on a specific remit (e.g. the performance of a particular product feature), then they are incentivized to avoid any work that doesn’t directly contribute to their remit.

### Chicken and Ping Pong
Another consequence of having multiple teams is that not all engineers can resolve all support tickets. This means that each ticket needs to find the engineer from the ‘right’ team. There are two systems that can be employed here: a ‘pull’ system where engineers pull relevant tickets from a shared pot, or a ‘push’ system where a triage team pushes the tickets to the correct team. Unfortunately, there are issues with both.

A pull system can result in “ticket chicken”, where individuals leave the ticket alone in the hope that someone else will pick it up because the ticket isn’t clearly in their remit, and they have other tasks they would rather do. A push system can lead to “ticket ping pong’, where a ticket bounces between many teams who all try to explain why it isn’t in their remit. Tickets that don’t have an obvious owner take longer to be picked up than those that do, meaning that they take longer to resolve. 

### Explicit ownership
An obvious solution to this problem is to force the ticket into the “obvious owner” bucket through explicit ownership. This is a very effective strategy, and one that gets you about 80% of the way there. We could apply this to the product owner scenario and resolve the issue by having a member of the team explicitly responsible for ‘helping with ad hoc requests’ at any given time. In a code base, you might want to have explicit owners for each technical component or service; this is easier in a microservices architecture than a monolith. There are many other benefits of explicit ownership, such as feeling empowered to make significant changes, or thinking strategically about a particular service.

### Problems with explicit ownership
While the explicit ownership model gets you 80% of the way there, it has some issues which are difficult to solve.

🔲 **You’ll never eliminate the grey areas.** If you try to ensure that everything in your product has an explicit owner, you enter a game of whack-a-mole from which you will never emerge. For example, who is responsible for keeping dependencies or your linting rules up-to-date?

📄 **Technical ownership documentation is hard to maintain.** It can get out of date quickly, and the more detail you include, the more overhead is incurred. If the documentation isn’t agreed as the source of truth, then you’re back to where you started with engineers disagreeing about who is responsible for what.

🙅 **It entrenches the behaviour of ‘it’s not my job’.** No one is encouraged to volunteer to take a task; instead, the cultural incentive becomes to argue about who should do the task. This obviously results in the tasks taking longer to resolve, but the culture can also spill into other areas of the business, and result  in bad relationships between teams. Additionally, all the time spent arguing is a wasted effort that could be better used elsewhere.

Despite these concerns, explicit ownership is a good approach to many problems, particularly issues around long-term code health. It can also help resolve imbalances caused by political dynamics in a team. But to be successful, we need to combine it with a “put your hand up” mindset.

### Put your hand up!
If this becomes part of the team’s culture, there are a number of benefits.
There will be fewer awkward silences in team meetings, and tasks should stop being assigned based on the principle of who has the least discomfort.
Tasks that don’t have an obvious owner will be picked up faster and with fewer arguments about ownership.
Team members will become more comfortable asking for help, as they are confident of getting a positive response.
Engineers will work on a wider range of problems and be more likely to step outside their comfort zone, and learn new skills.

This is also applicable between teams; if multiple teams start to behave in this way, then we see even more benefits.
If a problem arises without an obvious owner, teams will collaborate rather than waste time arguing about who should own it.
Engineers will start interacting more with the edges of the services they own, and better understand the way that different components fit together.
Team remits can become more fluid, which allows your company to flex its resources to better match demand without breaking up teams. By asking ‘who has time to do this?’ rather than ‘whose remit is this in?’ you can help avoid bottlenecks developing in under-resourced teams.

This all contributes to retaining a small company ‘we’re all pitching in’ mentality as you scale.

### Why don’t we all put our hands up?
‘We work hard to hire not just good engineers, but good people. Won’t this happen by itself?’
Unfortunately, this is a case of the prisoner’s dilemma. If only a few people put their hand up, they take all the burden of these tasks and may get frustrated and disheartened. Companies do not naturally incentivize this behaviour. It can be bad for someone’s happiness and career if they spend too much time away from their core role working on thankless, owner-less tasks. And worse still, they might get criticised for approaching a problem that is in someone else’s remit and “stepping on toes”.
For this to be effective, we have to change the incentives so that all our teams buy into this approach.

### How can I encourage this in my team?
For this to work in an organisation, there are a few prerequisites. Your team members will need:
to be able to allocate time to these kinds of task;
to be able to make prioritisation calls about their own work;
to be confident that they won’t get reprimanded for this behaviour (e.g. ‘you shouldn’t have offered to do that, it’s not your job’);
to know that their work won’t go unnoticed.

If I were writing a new set of company values, ‘put your hand up’ would be top of the list. It works well as a value because it’s easy to implement as an individual, and easy to assess as a leader. It’s something that you get for free in very small organizations, but it gets lost as a company grows, and teams struggle with ownership and capacity. If all team members had to write examples of ‘putting their hand up’ in their performance review every few months, there’d be a significant change in team dynamics.

### Conclusion
This isn’t just about software engineering. I used to work as a consultant for large, old-fashioned organisations and all the teams I worked with would have benefited from this approach. It’s also relevant in our personal lives: offering to take the bins out, or pick someone up from the station, is what our interpersonal relationships are built on.

*The takeaway message? Start putting your hand up.*

--

This was originally published on [LeadDev](https://leaddev.com/culture-engagement-motivation/put-your-hand-keeping-small-company-mindset-you-grow)