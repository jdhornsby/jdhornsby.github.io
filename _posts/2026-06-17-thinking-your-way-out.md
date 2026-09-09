---
layout: post
title: "Thinking your way out"
excerpt: "A follow-up to the schema-ordering experiment: raising a model's thinking budget closes the gap between good and bad field orderings, but the bad ones need two to four times the tokens to get there."
date: 2026-06-17 23:24:49 +0000
image: /assets/images/thinking-your-way-out/thinking-your-way-out.png
---

![](/assets/images/thinking-your-way-out/thinking-your-way-out.png)

In one of the levels the model generated, the final boss is weak to an item the player never has a chance to pick up. Every field reads as plausible and nothing is malformed, but the weakness points at something the player cannot be holding by the time they reach the fight, so anyone following the level hits a wall and stops.

A level can be fun or flat, and that is a judgment call, but a reference either resolves or it doesn't, and that is the only thing I judged this round. The prompt asks for something specific:

> 💡 Enemy weaknesses must reference the player weapon or a pickup from a prior encounter. The boss weakness must reference a specific pickup the player acquired during encounters. Each encounter trigger must set up the next encounter or the boss fight.

A level that violates one of those rules is broken in a way I can check, so this time I judged exactly that and nothing else: not whether the level was fun or coherent, only whether its references resolve. A level counts as broken if any encounter in it points at something the player can't be holding yet, and the share of levels with a broken reference is the break rate the charts below plot, where lower is better. A judge with one explicit rule to check is fast and close to ground truth, where the soft rubric I leaned on [last time](/autoregressive-schemas/) was neither.

None of this matters at my scale. The side project runs a handful of generations a day, and fifty percent more tokens or fifty percent fewer comes out as the same rounding error. The schemas aren't real either; they're six orderings of the same fields, built to crank field interdependency as high as it goes so that a small effect gets big enough to see. I pulled the thread anyway, because a schema handed to a model is a construct with a cost, and a construct with a cost is the kind of thing I check without deciding to, the same reflex that won't let me ship a query without reading its plan. It is part of the prompt, and right now it tends to get set by accident.

The earlier post showed that field order changes output quality, with the effect worst at minimal thinking. The obvious move is to turn thinking on, and this post is whether that works and what the budget costs.

## Re-judging across the swap

Google pulled the preview model out from under this between posts, which is [its own story](/preview-is-preview/), so I re-ran minimal on the stable release and put both minimal sets, old and new, through the new judge. The point was to confirm that the effect survives the swap and that the judge catches it in both sets, which it did. From there I walked the stable model up the thinking ladder, from minimal through low, medium, and high, at 72 generations per cell. The harness is in the [repo](https://github.com/jdhornsby/autoregressive-schemas).

The preview-minimal bar carries over as a reference to the first post rather than as a model comparison; there isn't enough here to say one release beats the other at minimal, and I'm not claiming it. Because the schemas are built to exaggerate, every error rate below is a ceiling, so the real-world numbers would run lower. And at 72 generations per cell the small differences are noisy, so the direction the numbers move is worth more than the size of any single gap.

## Thinking clears it

At minimal thinking the badly-ordered schemas break, the worst stable ordering failing on roughly one level in fifteen. The decision-ordered schema (nested_narrative in the table below) sits at or near the break-rate floor in every column and never needs the budget at all.

Turning thinking on closes the gap. The schema that was worst at minimal, append_order, reaches zero by medium and stays there; flat_alpha, less broken at minimal, turns out to be the stubborn one and doesn't clear until high. Across the bad orderings, medium and high drive the error to roughly nothing. More budget gives the model room to keep a later field consistent with an earlier one instead of committing blind, and on this task that is enough. What I can show is where the number went, not what the model did to get there, since I have error rates and token counts rather than thinking traces, so the mechanism stays a guess and I will leave it one.

One result I can't explain: low thinking did worse than minimal on two of the schemas, repeatably across reruns. A little thinking was worse than none. I don't have a story for it, and I won't invent one to cover the gap.

![](/assets/images/thinking-your-way-out/05_reachability_chart.png)

## What it costs

Thinking isn't free, and the cost barely moves with the schema. At high budget every ordering spends roughly the same number of thinking tokens, clean and broken alike, so a good schema doesn't earn cheaper thinking so much as it earns you the option of skipping the budget entirely, clearing the bar at minimal where thinking is zero.

That reframes the saving as something other than fewer thinking tokens. The saving is in never spending them.

![](/assets/images/thinking-your-way-out/06_thinking_tokens_chart.png)

Here is the cheapest setting that gets each ordering to roughly zero reference errors, by total tokens:

| Variant | Cheapest setting at ~0% error | Mean total tokens | vs cheapest |
| --- | --- | --- | --- |
| grouped_by_type | minimal | 573 | baseline |
| ui_contract | minimal | 579 | +1% |
| nested_narrative | minimal | 589 | +3% |
| alpha_nested | medium | 1,210 | +111% |
| append_order | medium | 1,218 | +113% |
| flat_alpha | high | 2,326 | +306% |

The clean orderings clear the bar at minimal, while the bad ones climb the ladder, most to medium and the worst all the way to high, paying two to four times as much to land in the same place. The task, the model, and the output are identical across the rows; what changes is the order of the fields, and that is the root cause that forces the bad schemas up the ladder, charged on every call. The cheapest ordering isn't even one I built to be good, which is the point worth keeping: clearing the bar takes nothing clever, only an ordering that doesn't work against the model.

At a handful of calls a day this stays a curiosity, but wrap the same schema in a tool surface that fires a million times and the order of the fields becomes a line on the bill that nobody chose. The specific saving I measured here is contrived, but the variable behind it sits in every structured-output tool, usually inherited from a REST API that ordered its fields for a database or a UI, neither of which is the thing generating against them.

## Scope and limits

One model, one task, six orderings built for dense interdependency, a single judge on a single property. Tasks with looser field dependencies will show less. The token figures are breakeven on a toy tuned to be hard, so treat them as a ceiling rather than a quote. The mechanism behind the compensation is unmeasured, error rates and token counts with no traces. The low-thinking dip is real and unexplained. Still in progress: whether the ordering effect holds on other small models without thinking.
