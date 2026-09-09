---
layout: post
title: "Preview is preview"
excerpt: "Google deprecated a preview model with almost no warning, prompting a harness-driven comparison against its GA successor on quality, reliability, latency, and cost before deciding whether to migrate a production workload."
date: 2026-05-24 19:20:23 +0000
image: /assets/images/preview-is-preview/preview-is-preview.png
---

![](/assets/images/preview-is-preview/preview-is-preview.png)

A preview model is a deal. You get something good early, you accept the rough edges like throttling, inconsistency, and surprise behavior, and you accept the clock. The deal is worth taking when the preview model is the best thing available for what you are doing. It is not worth taking blind.

Google has been particularly aggressive on the clock lately. They launched Gemini 3.1 Flash Lite Preview, deprecated it with almost no warning, and then shipped the stable version with a deprecation date one year out and no announced replacement. Born to die. In a year. I run a different side project on that model, and the clock is already ticking with nothing obvious to move to when it runs out. This is GCP's normal posture toward things you build on, and it is the reason this post exists.

Gemini 3.5 Flash went GA on May 19, and it is the obvious successor to Gemini 3 Flash Preview, which is what one of my side projects (a tiny playable adventure generator for kids) runs on today. Google has not announced a deprecation date for 3 Flash Preview yet, but the right time to decide whether to migrate is before they do. I was on the preview model because it was clearly better than Gemini 2.5 Flash at the task I needed: better outputs, more consistent across runs, worth the throttling I had to build resilience around. That is the only reason to take the preview deal in the first place.

## The harness

I do not swap models on a vibe. Before I touch production, I run my real workload against both models with a stronger model judging the output on a rubric, and I look at the numbers. The numbers in this post come from a public demonstrator of that approach, the [autoregressive-schemas repo](https://github.com/jdhornsby/autoregressive-schemas) from the [previous post](/autoregressive-schemas/), extended to 72 input cases. Same setup: the decision-ordered nested schema that won the earlier study, same prompt, same `thinking_level: minimal`, 144 generations per model, judged by Claude Sonnet 4.6 with extended thinking on a six-criterion weighted rubric. My actual production harness is structurally the same but written in a different language and using its own prompt, schema, and rubric. The shape of the validation is what matters, not the specific code.

## What the run showed

![](/assets/images/preview-is-preview/flash_v1_gemini_35_flash_vs_flash_v1_gemini_3_flash_preview_chart.png)

**Quality.** Mean rubric score 4.46 for 3.5 Flash against 4.10 for 3 Flash Preview, and 3.5 won 55 of 72 cases head-to-head with 17 losses and no ties. The gains concentrate on reference integrity, causal chain, balance, and mechanical sense, the consistency criteria, exactly where I want them for coherent generated worlds. Thematic coherence is flat. Completeness slightly favors 3 Flash Preview, and at this n I am not reading anything into it.

**Reliability.** Zero retries across 144 calls on 3.5 Flash. On 3 Flash Preview, 12.5% of calls needed at least one retry and the worst single call needed five. That's the difference between a preview model and a GA model showing up in the numbers, and it's also less resilience code I have to keep paying attention to.

**Latency.** Close enough to call even. 3.5 Flash is ~250 ms slower at the median but tighter, with about half the standard deviation and a worst case of 8.8 seconds against 12.9 for 3 Flash Preview. I will take consistent-and-slightly-slower over fast-and-jittery for a UI that already shows progress.

**Cost.** 3.5 Flash is 3x the per-token price of 3 Flash Preview, and produced ~33% more output tokens in this run, putting the real per-call cost ratio at ~4x. The token gap is partly whitespace: at `thinking_level: minimal`, 3 Flash Preview minifies its JSON about 40% of the time, while 3.5 Flash pretty-prints every time, which is what the bimodal distribution in the chart shows. Whether 3.5 Flash can be steered to minify is an open question I did not chase. For this product the absolute dollars are rounding error regardless.

## Migration

The code change was the model ID. The harness is why that was enough.

The point is not that you should run Gemini 3.5 Flash. The point is that the answer to "should I migrate" is not in the changelog, it is in running your own task against both models with the numbers you actually care about in front of you. Preview is preview. Build the muscle before you need it.

## Scope and limits

One task, one schema, 144 generations per model, one judge, one day. Quality on personalized game generation is a squishy thing to score and the rubric is mine. The bimodal output behavior on 3 Flash Preview is noted, not characterized; I did not try to steer it. No claim about 3.5 Flash being better than 3 Flash Preview in general, only on this task on this rubric. If you are on preview and considering staying, the right move is to run your own version of this against your own workload. The harness is in the repo.
