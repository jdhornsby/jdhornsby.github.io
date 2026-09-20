---
layout: post
title: "No need to sample"
excerpt: "LLM judges are slow and expensive, so you run them after the fact and score a sample. Jev is fast and costs a hundredth of a cent, so you can judge everything inline. I pointed it at a Haiku judge's dataset to see how well they agree."
date: 2026-09-20 20:00:00 +0000
image: /assets/images/no-need-to-sample/no-need-to-sample.png
---

![](/assets/images/no-need-to-sample/no-need-to-sample.png)

Yesterday I did some [initial exploration](/fifty-cents-of-jev/) of [Jev](https://docs.typesafe.ai/introduction). I tried a few use cases and was impressed by all of them except chess, which it played terribly, at least under my harness. But I wanted to try what I think is the killer use case: judging LLM output.

Alongside programmatic checks, it's common to use one LLM to judge the output of another. That comes with two constraints. A judge call adds seconds, so you run it after the fact, which is fine for a slow feedback loop and bad for catching a mistake as it happens. And a judge call is relatively expensive, so you score a few percent of outputs and assume the rest look like the sample. Basic economics.

In this post, I take a Haiku judge I'd already built and run, a [reachability check](/a-number-that-looks-fine/) on generated game levels, and point Jev at the same dataset. It's fast, it's dirt cheap, and it mostly agrees with Haiku.

I think the economics just changed.

## When you can judge everything

Jev obliterates both constraints. It can score a piece of content on multiple dimensions in a quarter of a second for a fraction of a cent. At that price there is no reason to sample. At that latency you can do it before sending the result to a user.

This is not an original idea. TypeSafe is pretty clear that this is what they built Jev for. They even named the model after the [economist](https://en.wikipedia.org/wiki/Jevons_paradox) who noticed that making something cheaper tends to mean you use far more of it.

Jev is well suited to use cases that require clear judgment calls about a piece of content. Does every claim in this answer have a citation? Does every citation point at a real source? Does this summary introduce a number that wasn't in the source material?

It is also competent at use cases where semantic meaning matters and the judgment is fuzzier. Is the tone right for a customer who is already angry? Does this response answer the question or restate it? Is this within policy, or does it need a human?

It does take some design work. My first attempt was a straight port of an old six-criterion rubric, and the scores came back compressed: the same rankings as the original judge, but with much less spread. I'd bet that's a me problem. Narrow questions with explicit criteria are what Jev is built for, and a rubric written for a generative judge isn't that. It's a different skill, and I'm still learning it.

The pattern is simple. Judge every important generation inline. Act when confidence is high, escalate when it isn't. Catch mistakes early.

## The numbers

I had a dataset from another experiment I ran a while ago. I was exploring the impact of schema design on structured output (my favorite topic lately). I had an LLM generate level configs for a fake game using six schema variants ranging from intentionally confounding to hopefully optimal. It included two types of judgments: a Likert-style score on six dimensions that tried to measure how good the level was and a more mechanical check on whether the end of the level was reachable.

I converted these judgments to a Jev `Score` and `Noul` respectively. Here is how they compare to the original Haiku results (n=64 levels per variant, three encounters each):

| Variant | Original Likert | Jev Likert | Original break % | Jev break % |
|---|--:|--:|--:|--:|
| flat_alpha | 3.84 | 4.46 | 5.2 | 3.6 |
| alpha_nested | 3.84 | 4.47 | 4.2 | 2.6 |
| ui_contract | 3.89 | 4.56 | 1.0 | 1.0 |
| append_order | 4.18 | 4.55 | 8.9 | 7.8 |
| grouped_by_type | 4.18 | 4.62 | 1.6 | 1.0 |
| nested_narrative | 4.32 | 4.64 | 0.0 | 0.0 |

Both judges rank the variants the same way, with nested_narrative clean and append_order breaking most often.

On the Likert-style rubric, Jev was more lenient and compressed. Zooming in, the two judges did have a positive Spearman coefficient (0.58 on the weighted total), so they generally ranked them similarly. In a real world scenario, I could try moving my threshold, but I think I'd redesign the rubric in this case.

The break percentage measures how many levels are not logically completable. These line up closely, with Jev a little more lenient on most variants. I was curious about where it differed, so I compared each and grouped them by what happened:

| Category | Verdict | Jev confidence | n |
|---|---|--:|--:|
| semantic match | Jev more accurate - Haiku missed it reading literally | 0.71–0.96 | 6 |
| plausible inference | debatable | 0.60–0.83 | 3 |
| semantic overreach | Haiku more accurate - Jev too generous | 0.50–0.67 | 2 |

The two calls Jev got wrong are also the two it was least sure about. Everything defensible starts at 0.71. That's the threshold behavior you'd want from a gate, and it's why this felt natural to use for this kind of check.

I then reran the full set 9 more times. Variant means held to within 0.006, though individual scores did move between runs (mean spread 0.11 on a 0–4 scale). It's stable in aggregate, but not per item. The reruns also gave me latency numbers:

![](/assets/images/no-need-to-sample/latency.png)

This is calling from my laptop on the East Coast. I'd love to get numbers with more favorable regional colocation and networking.

I spent $0.56 on the 5,866 requests I made in this test and related probing. That's about a hundredth of a cent per call.

## Scope and limits

One dataset, one domain, and one I built myself to stress exactly this kind of interdependency. The results are specific to my example. YMMV.

The comparison is against a Haiku judge, not ground truth, and where they disagreed I adjudicated the cases myself. The original judgments also have the problems I wrote about in [a number that looks fine](/a-number-that-looks-fine/).

All timings are from my laptop on the East Coast, against `jev-1.13`. I expect things will change rapidly.

Run it yourself: [repo](https://github.com/jdhornsby/typesafe-jev)