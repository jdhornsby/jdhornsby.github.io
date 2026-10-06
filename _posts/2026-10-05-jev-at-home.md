---
layout: post
title: "Jev at home"
excerpt: "Testing six small open Jev-style models for accuracy, confidence and self-hosting."
date: 2026-10-05 20:00:00 +0000
image: /assets/images/jev-at-home/jev-at-home.png
---

![](/assets/images/jev-at-home/jev-at-home.png)

In the weeks since Jev's release, a handful of open models inspired by it have appeared, along with [JevBench](https://benchmarkheaven.com/jev-models) to compare them. In earlier posts I tested Jev on a mix of public benchmarks and some private tasks of my own ([1](/fifty-cents-of-jev/), [2](/no-need-to-sample/)). I wanted to know how the open alternatives hold up.

The reason is pretty simple. TypeSafe is yet another service provider to integrate with. Jev is fast, cheap, and good. But getting it through a company's legal department might not be. And adding another subprocessor to the trust center is worth pausing on. If one of these models is good enough at my task and small enough to fit on cheap hardware, it might be a viable alternative.

So I pulled down half a dozen of the hottest Jev-style models (kev-0.8b, kev-4b, jeff-0.8b, strands-2b, decider-2b, and decider-4b) and ran them through my harness to find out.

The short answer is maybe: decider-4b looks good enough, and small enough, to be a practical alternative.

## Reachability

In an [earlier post](/thinking-your-way-out/) I built a judge for the levels my game generates. Given a level, can the player actually reach each encounter? It's a yes or no question per encounter with three encounters per level. It needs a little reasoning over structured data and some base level world knowledge. It is a great stand-in for a lot of real-world tasks I'd want a model like this for.

It is also private. None of these models could have trained on it. Here's the result:

![](/assets/images/jev-at-home/reachability.png)

I measured three things:

- **Accuracy:** how often it produces the correct answer.
- **Discrimination (AUROC):** is it more confident when it's right than when it's wrong? 0.5 is a coin flip; 1.0 means every right answer was more confident than every wrong one.
- **Calibration (ECE):** when it says 80%, is it right about 80% of the time? This is the average gap between stated confidence and actual accuracy, so lower is better.

A decision model like these needs to score well on all three. It needs to give the right answer, be confident when it's right, and have confidence numbers that mean what they say. This also has to be measured per task, even for Jev.

Jev leads on all three. decider-4b and kev-4b trail it but do well enough that they might be viable candidates. The smaller models each fall short somewhere: kev-0.8b gets 90% right but has the worst calibration of the group, jeff-0.8b is well calibrated but only 74% accurate, and both 2B models are barely better than a coin flip on accuracy. Jev's discrimination interval is wide, since it disagreed with the labels so rarely, so its lead there is real but less certain than it looks.

## Knowledge, routing, and hearsay

I also ran the public benchmarks from my earlier Jev posts:

- **MMLU:** a common test of world knowledge (n=14,033).
- **Routing:** classifying customer banking requests into intents, from banking77 (n=3,076).
- **Hearsay:** deciding whether a statement is hearsay, from LegalBench (n=94).
- **Likert:** the same game-level judge as above, scoring quality on a 1 to 5 scale (n=384).

Each cell is accuracy · discrimination (AUROC) · calibration (ECE).

| model | MMLU | routing | hearsay | Likert (exact / within 1) |
|---|---|---|---|--:|
| jev | 91.6% · 0.83 · 0.03 | 79.2% · 0.85 · 0.08 | 76.6% · 0.76 · 0.18 | 45% / 76% |
| decider-4b | 74.4% · 0.84 · 0.04 | 87.1% · 0.86 · 0.02 | 70.2% · 0.68 · 0.20 | 40% / 74% |
| kev-4b | 71.2% · 0.82 · 0.19 | 84.9% · 0.85 · 0.20 | 70.2% · 0.69 · 0.36 | 30% / 52% |
| decider-2b | 62.2% · 0.79 · 0.06 | 80.2% · 0.88 · 0.03 | 68.1% · 0.57 · 0.31 | 39% / 71% |
| strands-2b | 52.1% · 0.74 · 0.09 | 85.5% · 0.85 · 0.01 | 64.9% · 0.57 · 0.21 | 27% / 43% |
| jeff-0.8b | 50.9% · 0.73 · 0.07 | 63.8% · 0.79 · 0.10 | 59.6% · 0.60 · 0.29 | 25% / 45% |
| kev-0.8b | 44.9% · 0.70 · 0.15 | 81.0% · 0.86 · 0.13 | 54.3% · 0.53 · 0.27 | 18% / 45% |

Some takeaways:

- World knowledge shrinks with model size. No surprise there.
- decider-4b's confidence is as good as Jev's on MMLU and routing. It just knows less.
- The routing results are probably contaminated. banking77 is public and old, and the open models likely trained on it.
- Hearsay is a really small dataset, so it's hard to make much of it.
- kev-4b's confidence ranks answers well but is badly calibrated. It would need rescaling before you could set a threshold on it.
- The Likert judge is really hard. It was built for a frontier LLM, but decider-4b gets close to Jev within one point.

## But can they play chess?

No.\*

## Notes on hosting

If you're replacing a hosted API, the model is only half the job. The other half is serving it. The things I care about:

- **Footprint:** how much VRAM am I going to need?
- **Batching:** can it handle many requests at once, or does it process them one by one?
- **Serving stack:** does it run on something standard like vLLM, or does it need its own server?
- **Operations:** logging, metrics, health checks, and everything else you need to run it in production.

I haven't tried to answer these questions, but did peek at these models with this in mind as I was testing them. Here's how each looked:

- **decider** says it ships with a vLLM option. I didn't test it, but if it works as advertised, that's a big advantage: vLLM handles batching and memory for you, and it's a well-known stack to operate. decider-4b is also small, at 8.7 GB.
- **kev** comes with its own batching server, and it looked solid. I pushed kev-0.8b to about 120 requests per second on my RTX 3080 answering MMLU questions. But it's a custom server, so you'd have to build your own logging and metrics around it, and its architecture makes moving it to vLLM a fair bit of work. kev-4b is also bigger, at 14.3 GB.
- **jeff** was easy to adapt to vLLM. It took me a few hours, but that's out of scope for this post.
- **strands** wasn't fit for purpose, even locally. Requests were processed serially out of a queue. On my M4, I easily filled the queue and my client started getting timeouts. Abandoned requests stayed in the queue. Its architecture also would take some work to adapt to vLLM if that is your jam.

## Scope and limits

A handful of models. There are 92 on JevBench currently. I tested some well-known ones that fit on my hardware.

One run each. It is not a big sample so variance could matter.

Mixed hardware. Most of it I ran on an RTX 3080, but kev-4b was too big so I ran it on an M4 with MLX. Could shift the numbers a bit.

No tuning. Some prompt optimization or temperature scaling could improve the results.

More details in the [repo](https://github.com/jdhornsby/decision-models).

\* On chess: not with my harness, which is not designed to help the model at all.