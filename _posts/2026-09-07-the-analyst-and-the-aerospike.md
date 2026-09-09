---
layout: post
title: "The analyst and the aerospike"
excerpt: "The next generation of enterprise platforms will not put agents on the request path. They will let agents build things that get saved, reviewed, and run for years. The physics are the product."
date: 2026-09-07 20:00:00 +0000
image: /assets/images/the-analyst-and-the-aerospike/the-analyst-and-the-aerospike.png
---

![](/assets/images/the-analyst-and-the-aerospike/the-analyst-and-the-aerospike.png)

A while ago I read an article about a rocket engine designed by a machine. It was an aerospike and it looked nothing like what a human would design. It was organic and almost alien. Rather than design the engine, the team specified the physics it had to respect and the mission it had to fly. AI found the optimal shape within that space. And it worked. The design was done. The engine was printed.

Software doesn't work like that yet.

## A simple join

A customer wanted one data set created from two. Orders came from one system and shipments from another. The system could do this... technically.

The analyst working the ticket succeeded, of course. The configuration language let her craft a series of JSON patch operations - thirty or so - that combined the two result sets into exactly what the customer had asked for. It took half a day. An engineer given the same request would have written three lines of code and gone to lunch.

I have seen this scene play out many times over the years, in more than one company. There is always an analyst. There is always a language, a configuration, or a rules engine that cannot quite do what is being asked of it, and someone whose job is to get there anyway.

## The ceiling

Customers want infinite customization. Nobody has infinite engineers. That gap has been the central economic fact of enterprise software for as long as it has been sold. Configuration is the answer the industry landed on. Build a layer that expresses the common shape of what customers ask for, hire people who can work in it, and you have turned an engineering problem into a staffing problem. For twenty years that has been the right answer.

The platform underneath was usually fine. Its configuration exposed the APIs, used the right credentials, and enforced tenant boundaries. But the configuration layer was sized to the operator. It was something a smart person could learn in a few weeks without needing to be a programmer. That was the whole point. Everything its authors anticipated was easy. Everything else was a hoop or a ticket. The ceiling was never the platform. It was the person configuring it.

## The constraint is gone

The ceiling was a person who could not be expected to program. The cost of programming just collapsed. An agent can deliver code written against the platform's existing APIs that meets the customer's exact need. Another can deliver the tests that prove it works. Nothing has to be anticipated anymore because nothing has to be expressible in advance.

The investment moves from the configuration layer to the platform underneath. The sandbox the generated code runs in, the audit trail it produces, the observability that tells you what it did, and the processes that keep thousands of generated artifacts patched and current as the platform changes underneath them. This is a reallocation instead of a rebuild. The operators, credential handling, and tenant boundaries are still there and are still correct. What gets replaced is the layer that was standing between the customer's requirement and the platform's actual capability.

This is already being built. Legion is an open-source runtime where agents do work by writing code against tools you register, sandboxed with time and memory budgets, where the generated code never sees your credentials. Cloudflare and Anthropic both explored versions of the same idea over the last year. Give the model somewhere to write and run code, expose the platform as functions it can call, and let it compose. The pattern is called code mode and, for now, it is becoming the default way serious agents use tools.

## The interesting word is save

The people building right now are putting the agent on the request path. The user asks, the model figures it out, and the agent answers. When the user asks again tomorrow, the model figures it out again. Code mode makes this fast and cheap and is a step in the right direction, but it is still an agent doing the work every time.

The people building tomorrow will take the agent off the request path. The user describes what they want, the agent writes the code and tests, and the result gets saved. Then the user can use it again and again. It is just software. When it needs to change, the agent is ready to help.

Call these approaches the hot path and the cold path. The agent on the hot path is in the loop. The agent on the cold path is in the toolchain.

This is what I found so interesting about the aerospike. Yes, the machine found a strange shape, but then the search was over. It was a fixed object that could be manufactured and flown repeatedly.

Something you can save is something you can check. The tests come with it, they run before it ships and every time it changes after, and a human can look at it and reason about it. A hot-path answer cannot be tested, because there is nothing to test until it has already run.

It can also be explained. When a regulator asks why a decision came out the way it did, the answer is a file and a test suite, not a transcript of a model reasoning its way there in the moment.

And the economics invert. Hot-path inference is billed per interaction, so the bill scales with how busy the business is. Cold-path inference is billed per change, so it scales with how often the business changes. Those are very different numbers.

## The physics are the product

The obvious move for an incumbent is to put an agent on top of the configuration layer and call it done. It works. The analyst gets faster, the tickets clear, and the demo is good. But the thirty patch operations still get written and the ceiling is still where it was. Everything the authors did not anticipate is still a hoop or a ticket.

The shops already doing this for themselves are proof that it works, but they are not the market. They can do it because they already run tenancy, audit, and access control and have someone carrying a pager. Everyone else understands their own business better than any vendor ever will and has no way to build any of that. An accounting firm can describe exactly how it wants to handle its clients but it cannot build a sandbox. That gap is where the value is.

The analyst does not disappear either. The configuration language was only ever the medium she was handed. Her actual job was sitting with a customer who half knew what they wanted and turning it into something specific enough to build, and that work is untouched. What changes is what she hands over when she is done. Instead of thirty patch operations she produces a description precise enough for a machine to satisfy and a set of tests that say whether it did.

This is what I would build if I were starting a company today. Not because the economics are better, though they are, but because I want to see what comes out the other side. Every ticket that died on a roadmap was a customer with an idea that nobody could afford to build. When that stops being true, the things people make inside these platforms are going to be much stranger than anything on the roadmap was. That is what the aerospike team understood. They wrote down the physics and the mission and let something else find the shape. And the shape was nothing a person would have drawn.