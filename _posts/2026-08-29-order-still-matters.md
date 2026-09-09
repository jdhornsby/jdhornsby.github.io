---
layout: post
title: "Order still matters"
excerpt: "The first post in this series found that field order sometimes changes output quality, and it ended with a promise to check whether that holds on a small model that doesn't think, from a family other "
date: 2026-08-29 22:01:44 +0000
image: /assets/images/order-still-matters/order-still-matters.png
---

![](/assets/images/order-still-matters/order-still-matters.png)

The [first post](/autoregressive-schemas/) in this series found that field order sometimes changes output quality, and it ended with a promise to check whether that holds on a small model that doesn't think, from a family other than Gemini. The [last post](/declared-not-delivered/) was the failed version of that experiment, where Vertex alphabetized my schemas before the model ever saw them and the independent variable never made it out of the request. This is the run that worked.

## Dusting off the server

I have spent about a year using models through APIs, mostly Vertex and Bedrock, without paying much attention to what was running underneath them. That was fine until field ordering turned out to be something the serving layer might rewrite on its way past, at which point I wanted a stack I could completely control.

I tried the MacBook Air first, an M4 with 24GB, and it was too slow to be worth using for a run this size. So I went and turned on the machine in my home lab, an RTX 3080 with 10GB of VRAM, which had been powered off for almost a year. Half a day of patching and driver updates got it healthy again. I started with Ollama, found that it was also reordering my fields, and did not chase that any further than confirming it, because what I wanted at that point was fewer layers between me and the model rather than a better-configured one. Everything below came off that box running llama.cpp directly.

## Two models, no thinking

I ran Qwen3-4B-Instruct-2507 first and then Qwen3-14B with reasoning disabled, using the same six schema variants and the same 72 cases as the original post, judged for encounter reachability the same way as before.

![Encounter break rate, Qwen3 4B vs Qwen3 14B, no thinking](/assets/images/order-still-matters/4b-vs-14b.png)

The effect is there on both models, and on the 14B the gap between the best and worst ordering is close to fourteen-fold. Nothing changed between those runs except which order the fields were declared in.

The comparison I care about most is alpha_nested against nested_narrative, because those two schemas are identical in every respect apart from the order of their fields, down to the nesting and the field names. On the 14B one of them breaks better than a quarter of all encounters and the other breaks under three percent of them. Every other comparison in this series has had some structural difference mixed in with the ordering, and this one doesn't.

The ranking is not the same as Gemini's. There, append_order was the worst variant at minimal thinking and flat_alpha was near the bottom, while here flat_alpha is the worst by a wide margin on both models and append_order sits in the middle. What does hold across all of them is the other end of the chart, where nested_narrative and grouped_by_type are at or near the floor on Gemini, on the 4B and on the 14B.

## Room for thinking

Qwen3-14B is a hybrid checkpoint, so the same weights that produced the numbers above will also produce reasoning traces if you ask for them, which made the thinking run cheap to set up. It was not cheap in memory. Reasoning tokens live in the KV cache along with everything else, and finding room for them next to a 14B at Q4_K_M on a 10GB card took some arithmetic that renting the model by the token had spared me from ever doing. It fit, with less headroom than I would have liked.

![Encounter break rate, Qwen3 14B thinking checkpoint](/assets/images/order-still-matters/14b-thinking.png)

Thinking collapses the three worst orderings into single digits and leaves the two best ones roughly where they were, which takes the spread across the six variants from something like fourteen-fold down to under three. That is the same pattern I found on Gemini, and it means the same thing here: a bad ordering can be rescued by spending reasoning tokens on it, and a good ordering does not need them, so the saving is in never spending them.

One variant goes the other way. ui_contract is worse with thinking on than either non-thinking run, and it is the only one that does this. I don't have an explanation and I am not going to invent one. It resembles the non-monotonic behaviour I saw at low thinking budgets on Gemini, where a few orderings got worse before they got better, and I would treat both as unexplained rather than as evidence of anything in particular.

## So what?

Four models now, across two families, with thinking and without, and what survives all of it is fairly narrow. Which orderings are bad turns out to be specific to the model, so the worst variants from one model are not a guide to another. The orderings that are good stay good, and nested_narrative and grouped_by_type have been at or near the floor everywhere I have looked, with a gap to the worst variant on the same model that is wide enough to matter if you are shipping something.

So the advice from the first post holds, with a different reason behind it. Order your fields by decision structure, not because any particular alternative is universally bad, but because that ordering has been reliably close to the floor on every model I have tried, and it costs nothing to do.

## Scope and limits

The 14B is Qwen3-14B rather than an Instruct-only checkpoint, so this is not a clean comparison against the 4B and I am not making any claims about model size. Both are quantized to Q4_K_M, which is a difference from the full-precision Gemini runs. The thinking checkpoint truncated levels more often than the others, and the reachability judge responded by inventing the missing encounters instead of reporting the gap, so I patched it and re-ran the judging for that column only, rather than spend another six hours of judging per column on the two I had already scored. A small number of judge calls were genuinely ambiguous on manual review, all of them in alpha-ordered variants, which is too few to read a pattern from but worth mentioning. Encounter counts per variant, including the levels that came back short, are in the repo.

All of this is directional. There is one generation per case and no repeats, so the ends of the chart and the size of the gap between them are worth something, and the difference between two variants sitting a point apart is not.

Run it yourself: the harness is in the [repo](https://github.com/jdhornsby/autoregressive-schemas).