---
layout: post
title: "Fifty cents of Jev"
excerpt: "TypeSafe's Jev scores questions instead of generating text. I spent fifty cents of free credits running it against MMLU, intent classification, a hearsay task, and chess to see if the hype held up."
date: 2026-09-19 20:00:00 +0000
image: /assets/images/fifty-cents-of-jev/fifty-cents-of-jev.png
---

![](/assets/images/fifty-cents-of-jev/fifty-cents-of-jev.png)

[TypeSafe](https://typesafe.ai/) released a new "System One" model this week called [Jev](https://docs.typesafe.ai/introduction). Half a dozen friends had pinged me about it before I finished reading the docs. The general consensus was that it's a game changer, but nobody seemed to know why.

TypeSafe came out of stealth on September 15 with a [$40M seed round](https://finance.yahoo.com/technology/ai/articles/typesafe-ai-emerges-stealth-40m-190000776.html). One of the founders, Diogo Almeida, helped build RLHF and InstructGPT at OpenAI, the research that became ChatGPT. Jev is something completely different. Instead of generating text, it answers typed questions. You send it some state, the questions, and the possible answers. It responds with the answer it picked, a probability for every option, and a confidence score. It supports three question types (or primitives): Choice (pick an option from a list), Score (rate against levels you define), and Noul (evaluate a yes/no question).

The interface is really simple:

```python
from typesafe_sdk import Choice, TypeSafeClient

client = TypeSafeClient()

response = client.system_one(
    state="I've been trying to connect my Stripe account for 3 days and it keeps failing.",
    questions={
        "department": Choice(
            instructions="Which team should handle this",
            criteria={
                "billing": "Payment or subscription issues",
                "technical": "Bugs or integration problems",
                "sales": "Pricing or account questions",
            },
        ),
    },
)

answer = response.answers["department"]
print(answer.choice)         # "billing"
print(answer.probabilities)  # {"billing": 0.84, "technical": 0.159, "sales": 0.001}
print(answer.confidence)     # 0.596
```

The claims are bold. Input costs $0.042 per million tokens. Output is too cheap to meter. It can't hallucinate. It responds in under half a second. Check out the [launch post](https://typesafe.ai/blog/introducing-system-one-models-and-jev).

I wanted to see for myself. I joined the waitlist, pleaded my case, and got in. Then I spent about fifty cents finding out what it's like.

## No benchmaxxing, please

Getting up and running with their [Python SDK](https://github.com/typesafe-ai/typesafe-sdk-python) was easy. I ran a couple of their examples and they worked fine. It really is as fast as they claim.

TypeSafe is [against benchmaxxing](https://typesafe.ai/blog/antibenchmaxxing). Their argument is that once a public benchmark gets optimized for, it stops telling you much, and that users should run their own evals. That makes sense to me, but I wanted to quickly get a feel for it, so I ran some common public ones. This is not meant as a comparison against other models, so take it for what it's worth.

### MMLU

TypeSafe says Jev isn't an LLM, but I wondered how much it knows about the world, so I started with all 14k multiple-choice questions in [MMLU](https://huggingface.co/datasets/cais/mmlu). It's a decent proxy for how much a model knows innately. It turns out Jev knows quite a lot, and it got through the whole dataset in four and a half minutes (at a concurrency of 12).

| category | accuracy | n |
|---|--:|--:|
| STEM | 94.7% | 3014 |
| Social sciences | 93.3% | 3074 |
| Other (business, health, misc.) | 91.7% | 3241 |
| Humanities | 88.4% | 4704 |

It had a few weak spots: college chemistry (75%), global facts (74%), and virology (56%). Let's zoom in on virology vs. a stronger subject:

| subject | n | accuracy | confidence when correct | confidence when wrong |
|---|--:|--:|--:|--:|
| high_school_mathematics | 270 | 95% | 0.88 | 0.46 |
| virology | 166 | 56% | 0.92 | 0.87 |

I am not faulting it here, but notice how confidently wrong it was. I suspect problems with the dataset, but it highlights the importance of testing your own datasets and not blindly accepting its confidence score.

### Intent classification

Next, I wanted to try something closer to what I'd actually do at work: intent classification, like routing a prompt to a specialist agent. [Banking77](https://huggingface.co/datasets/mteb/banking77) is 3k customer banking queries, each labeled with one of 77 intents. Jev routed all of them in under a minute.

| correct | confidence when right | confidence when wrong |
|--:|--:|--:|
| 79.2% | 0.92 | 0.70 |

That's a strong result. Some of the intents are nearly identical. "Card arrival" and "card delivery estimate" are separate labels, and Jev mixed them up with high confidence.

### Policy checks

Finally, I wanted to check if it could identify a policy violation. The closest public proxy I found was the hearsay task in [LegalBench](https://huggingface.co/datasets/nguha/legalbench). I gave Jev the rule and asked whether each of 94 scenarios is hearsay. It got 76.6% in three seconds.

| slice | n | accuracy |
|---|--:|--:|
| Non-assertive conduct | 19 | 100% |
| Statement made in-court | 14 | 100% |
| Not introduced to prove truth | 20 | 85% |
| Standard hearsay | 29 | 66% |
| Non-verbal hearsay | 12 | 25% |

It was perfect on the easy calls and struggled with non-verbal hearsay, where a gesture or action counts as a statement. That takes an extra step of reasoning, which TypeSafe lists as a [known weak spot](https://docs.typesafe.ai/model-jaggedness/jev-1.13).

## But can it play chess?

TypeSafe's [Doom demo](https://typesafe.ai/blog/introducing-system-one-models-and-jev) got me curious about how well Jev reasons about a world. I was also curious how it compares to [guided decoding](https://jdhornsby.com/nothing-illegal-happened/) for chess. Jev picking from a list of legal moves is the same constraint with a different mechanism. Guided decoding builds a move one token at a time and masks out anything illegal, which [skews the model's distribution](https://arxiv.org/abs/2504.09135): it commits early to tokens it likes, then gets forced into whatever legal continuation is left. Jev scores every legal move at once, so it doesn't have that problem.

I started with the board representations LLMs do ok with (and some they don't): SAN, ASCII, FEN, and PGN. It did badly across the board. I reread the Doom writeup and designed two more, JSON and prose. Those helped, but it still clearly struggles. That's not a surprise. TypeSafe's [FAQ](https://typesafe.ai) says chess-like planning may be better left to reasoning models.

Here's an entertaining game where it gets destroyed by Stockfish at 1320 Elo:

<video controls loop muted playsinline width="100%">
  <source src="/assets/videos/fifty-cents-of-jev/game.mp4" type="video/mp4">
</video>

I didn't expect it to do well. Chess is hard. It still left me unsatisfied, though. I feel like there's a representation of the board that would make it click, and I didn't quite get there. Others have gotten [further](https://dev.to/maximsaplin/typesafe-jev-played-chess-and-landed-next-to-reasoning-models-28ga) with [different harnesses](https://github.com/wondertwins/jev-benchmark), so I may revisit it.

## Game changer

So, is Jev a game changer? I think so.

There are so many natural places to use it right away. Intent routing and policy-based guardrails are two I explored above. The really interesting one to me is online evals. It's common to sample live generations and have an LLM judge score them, often alongside deterministic checks. Jev looks like a natural fit for that, and TypeSafe [pitches it for exactly this](https://typesafe.ai/blog/introducing-system-one-models-and-jev). But it's fast and cheap enough that you don't have to sample. You could score every generation, inline, before it reaches the user. That's the [Jevons paradox](https://en.wikipedia.org/wiki/Jevons_paradox), which the model is named after: make judgment cheap enough and you start putting it everywhere.

That said, it isn't magic. It can't hallucinate in the sense that it can only pick from the answers you give it, and TypeSafe is upfront that this is what they mean. But it can pick the wrong one, confidently, like it did on virology. Wrong answers just look like normal answers.

The confidence score is the tool for catching those, but it isn't a probability, and it doesn't come with a cutoff. You have to pick one. And that means rigorously testing your own task on a labeled dataset first.

How you represent state made a big difference in chess. It took time to develop "prompt engineering" intuition for LLMs. There is a similar skill required here, and it will take time to build.

This whole experiment cost $0.50 of the $5.00 in credits TypeSafe gave me. I have plenty left, and I plan to explore the judge use case next, after I finish some of my guided decoding experiments. This was a bit of a detour.

## Scope and limits

This was not a rigorous evaluation. I ran everything on `jev-1.13`. I expect things will change rapidly.

Every benchmark was a single run. I didn't tune anything. The goal was just a quick baseline. All of the timings are from my laptop on the East Coast.

Run it yourself: [repo](https://github.com/jdhornsby/typesafe-jev)