---
layout: post
title: Estimation is a Trap
date: 2024-08-24 19:04 -0500
---

# Estimation is a Trap

As a software developer, you'll often be given a list of requirements, immediately followed by, "When will it be done?" It's a perfectly reasonable question! The person asking probably has a boss and needs to provide an answer when they are asked the same question, because, well, their boss is asking the same thing. Additionally, knowing when something will be done helps others prepare for the next step in the software’s lifecycle. Again, all reasonable!

We, as problem solvers, have tried to come up with systems to predict the future using Agile methodologies, points, and burndown charts, as if a million different variables aren’t at play all at once. We've even turned it into a [nice game](https://en.wikipedia.org/wiki/Planning_poker)! Or, instead of a number, we can use [garment vernacular](https://asana.com/resources/t-shirt-sizing)! How fun!

## So, why is it a trap?

Let’s discuss underpromising and overdelivering. It’s a combination that often goes well together. However, when tasked with estimation, it’s very easy to slip into the realm of overpromising and underdelivering. It’s not that you mean to; you had the best intentions after all. It is a perfectly natural tendency for humans to do this, not just with time but also with time's close relative, money. Sometimes, both time and money are improperly estimated, especially in [government projects](https://en.m.wikipedia.org/wiki/Big_Dig). If you maintain a home, you know how an expert can come in, give an estimate, and then blow right past it significantly.

Some people are well aware of the [sunk cost](https://en.wikipedia.org/wiki/Sunk_cost) fallacy and will use it to their advantage. In Robert Caro’s ["The Power Broker: Robert Moses and the Fall of New York"](https://en.wikipedia.org/wiki/The_Power_Broker), there are stories where Moses would estimate a fraction of the true cost of a public works project. He’d start the project knowing full well that legislators wouldn’t want an empty hole where a highway should be when the initial funds ran out.

## Well, can I just not give an estimate?

Ah, yes! The only way to win is not to play! Unfortunately, you will be viewed as stubborn and unhelpful. I once worked with someone who would cross his arms, put his nose in the air, and state that he _simply wouldn’t estimate_. My dude, it’s not like the person asking is doing it for fun! Show some empathy and work with your comrade, alright? If you don’t provide the estimate, sometimes the person asking will just come up with their own, which is usually based on fantasy, but there’s still an expectation that you’ll meet that made-up deadline. No one wants that.

## But why are we so bad at it?

**Bias, ego, and a willingness to please.**

For those of us with a breadth of experience, when given a task, we instantly reach back into our memory banks to remember if we’ve dealt with a similar task before to use as a baseline. Our [own biases](https://en.wikipedia.org/wiki/Rosy_retrospection) color this measurement, and other factors might have changed, impacting your progress. Plus, we think we’re really good at our jobs. So good that this time will be a _breeze_.

If you don’t have that breadth of experience or if you feel uncertain, you might blurt out an estimate that you think will impress the person asking. It’s okay! We all do it. Being fast is often equated with superior skill. If you give a quick estimate and then meet it, it just proves how awesome you are.

These factors lead to you not being able to say the truth, which is that you don’t know for certain. If you’re just starting out or feel insecure in your job, showing those cards feels like a weakness.

## So, what’s the plan?

There are a few tools I use to try to give the best possible answer when put on the spot, but some of them won’t work if you don’t work in a safe and trusting environment. That’s a whole other issue I haven’t solved yet but fortunately don’t have to deal with currently.

### Confer

The act of pointing tickets is essentially meaningless. We all say it’s not based on time, but you and everyone else knows that it is. The real value in pointing is discussing with your peers what you need to accomplish and how you might approach it. They might have a better solution, know of potential traps, or ask clarifying questions. As a more seasoned developer tasked with estimating as part of being a lead, your guess is based on what it would take **you** to accomplish the task, not accounting for various levels of experience and skill. Team members of varying skills pointing allows you to remember that, as long as they feel comfortable expressing their real number. The downside of this approach is that it takes time and requires context switching for your team. It also obviously doesn’t work if you’re alone on your team.

### Spike

If you work in an environment where you can say, “I’m not sure, but I can quickly gather some info for a better guess,” this can also lead to a better estimate. If someone is asking you to do something you’re not certain about, determine a set time to do some research to gain knowledge about whether the task is even possible. This requires looking into your own code, documentation, institutional knowledge, and resources like Google, Stack Overflow, and blogs. Try to gauge how complex it is based on what others have gone through to accomplish a similar task, bearing in mind that their experiences may also be biased.

### Discuss Tradeoffs

Perhaps there are three easy parts of the task, but one part is unknown, such as a wild animation or a new navigation style or technology you aren’t familiar with. Have a discussion with the person asking. Maybe they are flexible about it. Don’t just say, “No, I won’t do that.” Explain that it’s an unknown but offer a solution. Perhaps the wild animation is not crucial, or a time-tested navigation style might suffice. Just discuss it and let the person asking know the cost associated with what they’re requesting. After all, they don’t know, which is why they are [asking you](https://xkcd.com/1425/).

### Fudge

If you don’t have any of these luxuries or if you work somewhere that doesn’t understand that you’re making your best guess, then take the time that is in your head and **double it**. This gives you a buffer for a busted code base, sudden requirements, sickness, and other unknown obstacles. I understand this takes confidence because you might fear that stating a longer time will result in being replaced by someone who can meet the original expectation. However, if they balk at the extended time, try discussing what could be trimmed so you can be more confident in meeting the revised deadline. _“But what happens if you finish in half the time? Won’t that hurt your integrity?”_ If it’s extremely egregious, yes, but generally, the person asking will be thrilled that it’s ready, and that will outweigh any concerns. Most of the time, issues arise, and you'll be glad to have that padding.

### Constantly Communicate/Document

Even if you have the luxury of using any of the aforementioned tools, but especially if you don’t, it’s in your best interest to communicate your current status to the requesting party. Show progress, communicate roadblocks, discuss how incoming requirement changes might impact the timeline, and if something that seemed easy turns out to be challenging, explain why and offer alternatives. Document everything because what tends to happen is that the person asking stops listening after receiving an estimate, **especially** when new requirements come in after the estimation. Beware! Even with after doing all that copious communcation, I have been bitten when I didn't meet a date I set months ago. If you have the documentation, it can come off as "I told you so" or blamey so tread lightly!

## Embrace the Suck

In my opinion, estimation should be a combination of how long you think it will take and how certain you are about it. If you don’t feel confident and the requesting party wants a concrete date, you should be given time to shore up any blind spots.

I like to use the analogy of cooking when explaining this to people who insist on a specific date:

I ask them how long it would take to make a peanut butter and jelly sandwich and usually get an answer like “five minutes.” Then I ask, “What if your house were on fire, you were missing butter knives, and the bread was moldy? What if, instead of a peanut butter and jelly sandwich, you had to make [fermented shark](https://en.wikipedia.org/wiki/Hákarl)? What’s the estimate then?” This is somewhat analogous to being a software developer. But to extend the analogy, the person asking is usually the waitstaff, and the customer is really hungry, so understand that they are just doing their job and it's all part of the system.

In a completely new app with a logical and sane API to interface with, I could spin up a table view in iOS within twenty minutes. But that's not the world we live in. We work within imperfect code bases, interfacing with wacky APIs, dealing with scenarios not anticipated, while working with underdeveloped requirements, and handling shifting desires. I wish we could not only state how long it would take but also how certain we are about it to provide an over/under estimate. But people generally care about the date and not about how we feel about it. In the meantime, use the techniques above to highlight the hazards and complexities you’ll need to work around. If you know any other techniques, drop me a line!


