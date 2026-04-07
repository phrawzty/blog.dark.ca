+++
title = "Devopsdays Berlin 2018; recap"
date = 2018-09-18
+++

The sixth edition of [DevOpsDays Berlin](https://www.devopsdays.org/events/2018-berlin/welcome/) was held over the 12th and 13th of September 2018, and despite the fact that it's one of the most consistent, longest-running Devopsdays events on the calendar, this year was my *first ever* opportunity to attend. The tl;dr is that this was a very fine event, and the local organising team clearly has their act together (though they are [actively recruiting](https://www.devopsdays.org/events/2018-berlin/contact/) for new volunteers).
The talks were not recorded, so for the benefit of those who couldn't make it, I've refined my notes from the talks on Day 2 (unfortunately I had to miss Day 1).

### Dirk, on trust

The day started with a presentation from [Dirk Lehmann](https://twitter.com/doergn) of SAP entitled "Trust as Foundation of DevOps". Dirk has been a DevOps practitioner and booster for years, and I've had the pleasure of seeing him speak a handful of times before - as usual, he didn't disappoint. The central thesis of this talk is right there in the title: that trust, that fickle, difficult, human emotional thing, is the fundamental element of DevOps. He proposed a few ways to think about trust beyond a philosophical concept, and introduced a way of measuring the level of trust within an organisation: by measuring *speed*. Fast decision making, execution, and turnaround are hallmarks of a trust-based organisation. In terms of fostering a trusting environment, he proposed three key elements: managing team size, cultivating diversity, and embracing "positive failure culture".
Regarding team size, he looked to the seminal Mythical Man Month, as well as the work of evolutionary psychologist Robin Dunbar (see: Dunbar's Number). Dirk's conclusion? 15 people is the largest a team can get before it it breaks - and often it should be smaller. Concerning diversity, the take-away was that teams should organise around projects, products, or goals, instead of the traditional "business unit" (or work-type silos) that have been commonplace for much of the post-industrial era. Finally, citing Google's research that investing in psychological safety - more than anything else - was critical to making a team work, Dirk argued that embracing failure as a learning experience is paramount.

### Yuwei, on mentoring

The next talk was by [Yuwei Ren](mailto:yuwei.ren@facebook.com), a Production Engineer at Facebook, entitled "Mentorship: From the receiving end". Despite the title, most of her talk was about mentorship from the perspective of the mentor, and it was chock full of good tips for anybody engaging in a mentoring relationship.
For example, as a mentor, how do you get off to a good start with your mentee? Start by establishing career goals, and then ask "How can I help get you there?". Ensure that the expectations are set, and that outcomes are clearly defined by both parties. This is done by establishing a plan to achieve those goals, and setting up milestones along the way. During the process, schedule regular check-ins, and stick to them! Set the bar high, and along with those lofty expectations, make sure that both parties are accountable to each other. The mentor/mentee relationship is a big responsibility, and it must be treated with respect and commitment.
Both parties must acknowledge that disagreements and misunderstanding are inevitable, so *empathy* is key. Assuming good intentions is important, but that's only the first part of the equation - both parties need to actively listen and respect each other both professionally and as human beings. Finally, the mentee needs to take responsibility for their career, actively seeking and providing feedback, and acting on the advice of the mentor.

### Ken, on DevOps transformations

Next up was [Ken Mugrage](https://twitter.com/kmugrage) of Thoughtworks (and fellow member of the DevOpsDays global core) with a talk entitled "Want to successfully adopt DevOps? Change Everything". His talk was *very dense*, and to be honest I didn't quite capture all of it. His four key points to successful DevOps adoption were: redefine words, change your organisation, change your architecture, and use CD to *safely* deploy more often.
Within a team specifically, and an organisation in general, agreeing on specific definitions is critical. In terms of defining the word DevOps, Ken argued that DevOps should not define a toolset, nor a role, nor a team. Instead, DevOps is *verbs*: developING and operatING. If you need a noun, why not CAMS or CALMS (upon which much has [been written](https://duckduckgo.com/?q=cams+or+calms+in+devops) before). Ken then offered his interesting description of DevOps:
> A culture where people, regardless of title or background, work together to imagine, develop, deploy, and operate a system.

Words are important, but action is critical, and the best way to implement CA(L)MS is to change how your organisation is, well, organised. Ken invoked [Conway's Law](https://en.wikipedia.org/wiki/Conway%27s_law) and by way of illustration, explained how just renaming the Ops Team to DevOps team isn't going to help (nor is any other action that just leaves silos in place). Teams need to be organised differently, which means that the whole organisation might need to be organised differently as well. Furthermore, the success of those teams must be measured by how they deliver *business value* (not just whether their software functions or not).
Ken then moved into continuous delivery versus continuous deployment, and this is where things got fuzzy for me.

### Sponsors!

Three sponsors had an opportunity to say a few words on stage: [Chef](https://www.chef.io/), [Google Cloud](https://cloud.google.com/), and [Quest](https://www.quest.com/). I mention this because it's important to acknowledge the important role that sponsors play in the DevOpsDays series - without them, the events simply wouldn't be able to happen. Thanks, sponsors!

### Tiago, on Microsoft

The final full presentation of the day was from [Tiago Pascoal](https://twitter.com/tipascoamsft) from the Azure DevOps Customer Advisory Team, speaking about how a unit within the Microsoft empire experienced its own DevOps transformation. This one started slow, but once Tiago got going, it ended up being *very* interesting - so I'll skip right to the good stuff.
Within the Azure DevOps unit, teams are assembled into 10 to 12 person groups, the composition of which is heterogeneous and organised around products (see: Dirk's argument above). These teams are *self-forming*, so everybody decides which team and/or manager they want to work with. Apparently, while 100% of the group gets a choice, typically less than 20% choose to change from their existing situation. (To me, this is *wild*, and I'd be very interested to try something like this out at some point in my career.)
Tiago invoked Daniel Pink's popular book Drive, noting that need teams 3 things: autonomy, mastery, and purpose. He added another layer, which he referred to as "Alignment vs Autonomy". Autonomy is everything related to planning and execution,  whereas alignment is the organisation, the team structure, and most importantly, the *cadence* of work. For example, they work in 3-week sprints, and at the end of the sprint, whatever is in the master branch gets deployed - so be ready.
At an organisational level they track metrics; but NOT the classics like burndown, velocity, original estimate, completed hours, team capacity, or number of bugs found. Individual teams can track those if they wish, but at a high level there's only one metric that matters: impact. Are they delivering value? Speaking of measuring, in terms of code quality, Tiago explained the "Bug Cap": `n * 5 = x` where `n` is # of engineers, and `x` is maximum bug limit. If the bug count exceeds the bug cap at any time, the team *stops* working on new features and moves immediately to stabilise. Neat!

### Ignites

After lunch, it was time for everybody's favourite DevOpsDays tradition: the **ignites**! I was preparing for my own ignite so I missed the [first one](https://www.devopsdays.org/events/2018-berlin/program/suresh-chandra-bose/) (sorry).

### Daiany, on relaxing at speed

This was an excellently-delivered lightning that told the (true?) story of two teams and their relative stress levels heading into the GDPR cut-off date. It was fun, light-hearted, and carried a strong message: when you're doing the DevOps at high velocities, even massive EU legislation can't stress you out! Well done, [Daiany](https://twitter.com/DaianyMargarita).

### Daniel, on serverless

[I](https://twitter.com/phrawzty) did a lightning talk on serverless. It was fun. :)

### Thanks!

First off, thanks to the local DevOpsDays Berlin volunteer group. Also, thanks (again) to the sponsors. Finally, thank-you to my employer, [Datadog](https://www.datadoghq.com/), for sending me out there to speak. There are so many moving parts to make even a small conference happen.
I hope to make it out next year!