---
layout: post
title: "A number that looks fine"
excerpt: "A weighted rubric judge can return a steady, reasonable-looking score on output that is badly broken. There are two independent reasons why, and neither one shows up in the score itself."
date: 2026-07-19 17:05:27 +0000
image: /assets/images/a-number-that-looks-fine/a-number-that-looks-fine-1.png
---

![](/assets/images/a-number-that-looks-fine/a-number-that-looks-fine-1.png)

If you have measured the quality of LLM output at any scale, you have probably built one of these. You write a rubric of a few criteria, you have a strong model score each one from 1 to 5, and you combine the scores into a weighted average. It is the realistic way to grade open-ended output when there are too many outputs to read by hand and no single right answer to check against. I have written this kind of judge many times, for work and for side projects.

It is easy to build and easy to trust. You write the criteria, you run it, it returns numbers, and the numbers look reasonable. Nothing about using it tells you when it is wrong. And a weighted average has two ways of being wrong that never show up in the number it hands you.

I found both in a judge of my own, so the example below is a game-level generator. The lesson is not about games.

## **A score that survives a broken artifact**

The judge was scoring generated levels. Each level has the player fight three encounters, and each enemy has a weakness the player has to exploit with an item. The catch is timing: the item for an encounter is only picked up after that encounter is won, so a weakness has to point at the starting weapon or at something from an earlier fight. Point it at the encounter's own pickup and the fight is unwinnable, because the player cannot be holding that item yet.

One level had all three enemies weak to their own encounter's pickup. Every fight unwinnable, the whole level impossible to finish. My judge scored it 4.75 out of 5.

That number should not be able to exist. Getting to why it does took two passes at the problem, and the two reasons are different.

## **The first way: the weights hide it**

To check the rubric I built a second judge that did one thing. It read each level and asked, per encounter, whether the weakness could be satisfied by something the player already had. One rule, a yes or no, close to ground truth. Over the same 64 levels the rubric judge averaged 3.95. The one-rule judge found that one in seven levels had at least one unwinnable encounter. Same outputs, and one judge said fine while the other said a seventh of them could not be completed.

The rubric had six criteria. Two of them, the ones most likely to notice an unwinnable level, I had weighted at the bottom, one point each out of twelve. The criteria that a broken level can still score well on, theme and the like, I had weighted higher. So even when the rubric noticed the problem and marked the relevant criterion down, the score climbed back up on the criteria that did not care about it, and landed near the middle. The one criterion that saw the defect was outvoted by the ones that didn't.

The honest version of how the weights got that way is the useful one. I did not weight them carelessly. I decided early that some criteria mattered more than others, which was reasonable on its face, and then never checked it. This was a demo, not something anyone depended on, so I set a plausible weighting and moved on without confirming it scaled the judgment I actually wanted. The weighting wasn't thoughtless. It was unvalidated, which looks the same from the outside and lands in the same place. And the same move, a sensible early weighting that never gets re-examined, walks straight into real systems where the stakes are not a demo. That is the first failure mode, and it is the tamer one: the judge saw the problem and the arithmetic buried it.

## **The second way: the criterion drifts**

The tamer one assumes the judge saw the problem. Often it doesn't, and it doesn't fail the same way twice.

I ran the rubric judge over the levels twice, two independent passes. On the broken level above, one pass scored it 3.33 and the other scored it 4.58. Same level, same rubric, two and a half points apart. On the low pass it caught the timing problem and marked the reference criterion down to a 2, noting every enemy was weak to a pickup from its own encounter. On the high pass it called the identical pattern a minor issue, scored that same criterion a 4, and wrote that no names were wrong.

The reason is in the rubric. The reference criterion asked whether names existed and matched. It never stated the rule that actually defined the failure, that a weakness has to point at something the player already has. So when the judge met a level built entirely on that failure, it was grading against a criterion that never ruled on the question, and it improvised. One run treated the timing break as fatal. The next treated it as a stylistic quirk worth a point. Both are defensible readings of a criterion that was never fully written.

This is the failure mode worth sitting with. It is not bias, and it is not the lossy average, though both are real here. It is that a criterion you left underspecified gets interpreted differently every run, and the overall score hides which reading you got. One pass caught the defect and the weights buried it. The next pass never caught it. The average looked about the same either way. You cannot tell, from the number, whether the judge saw the problem and discounted it or missed it entirely.

## **Why this is easy to miss**

Put the two together and the trap is clear. A weighted rubric is the only practical way to score a lot of this output, so you build one. It returns steady, reasonable numbers, so you trust it. And the number is steady even when the thing underneath is not: a real defect can sit inside a mid-range score, and a run-to-run coin flip on an underspecified criterion averages out to something that looks stable. The output that should alarm you looks exactly like the output that shouldn't.

The weights you set by feel and the criteria you left half-written are deciding what the judge can see, and the aggregate will not tell you what they decided. A clean score from a judge you have not checked is the most dangerous thing in the pipeline, because it is indistinguishable from a clean score that is earned.

The first thing to say is that a lot of what a rubric judge does badly should never have been the model's job. Reachability is a rule. A weakness has to reference the starting weapon or an earlier pickup, and you can check that in code that walks the inventory and compares names, no model in the loop, the same answer every run.

Anything you can write as a rule, write as a rule, and let the code judge be right the way the rubric judge can't. My second judge was an LLM only because I wanted a loose match on item names and didn't want to spend the afternoon on the exact version; for a real system I'd have written the check, and it would have caught every broken level here.

The model judge is for what genuinely resists a rule: whether the level is fun, whether the tone holds. There a rubric is the only thing that scales, and scale is exactly why it has to be calibrated. Most of the published work on judge reliability goes after bias in the model. Very little goes after the step where you crush a rubric into one number, which is where mine actually broke.

The habit that survives all of it is dull and it works: take a handful of outputs you already understand, and see what your judge says about them. If it disagrees with you on the cases where you know the answer, the number it gives you on the cases where you don't is worth nothing.

## **Scope and limits**

One rubric judge against one targeted judge, over 64 generated levels, scored twice each. The one-in-seven break rate is a fact about this set, which was built to stress exactly this kind of interdependency, not an estimate of how often it happens anywhere. The rubric judge's swing across passes is large, and that is part of the point, not a caveat around it: an instrument that scores the same broken artifact 3.33 and 4.58 is telling you what its number is worth. The judgments and both prompts are in the [repo](https://github.com/jdhornsby/autoregressive-schemas). The useful move is not to trust any of this. It is to run your own judge against cases you already understand and see whether you believe it.
