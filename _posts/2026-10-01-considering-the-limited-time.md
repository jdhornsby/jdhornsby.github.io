---
layout: post
title: "Considering the limited time"
excerpt: "I expected thinking models to commit to their chess moves. They mostly wanted two newlines."
date: 2026-10-01 20:00:00 +0000
image: /assets/images/considering-the-limited-time/considering-the-limited-time.png
---

![](/assets/images/considering-the-limited-time/considering-the-limited-time.png)

In the [last post](/nothing-illegal-happened/), I built a guided decoding harness that makes language models play chess. The guide gets legal moves from the chess engine running the game and constrains the model to only emit legal moves. While the model can never make an illegal move, that doesn't mean it wants to make a good move or any move at all. The harness measures the difference between what the guide allows and the probabilities the model assigns to outputs.

Note, chess is just a fun example to explore the effect of guided decoding. I don't expect the models to be very good at it. Other people have built harnesses optimizing for that. My goal is simply to better understand the interaction between model and guide.

In this post, I want to add thinking to the mix. I expected that, if the model reasoned about the board, it would better commit to a move and I would see that in the output. It did not turn out that way, at least not at first.

## Adding thinking support

A thinking model starts its turn by reasoning in free text, then emits an end-of-thinking token, then answers. The reasoning should not be constrained by the guide. How could it be? We need to make the guide skip it somehow.

The approach is simple. The guide stays disengaged from the start of the turn until the model emits `</think>`, and then it takes over and constrains the rest of the response. I also wanted to keep turns from running forever, so I added a thinking budget. If the model reaches the thinking budget limit without finishing, the harness injects `</think>` itself and forces the model to answer.

```python
def generate(model, guide, ids, think_end, max_think_tokens):
    thinking = True
    thought = 0

    while True:
        logits = model(ids)
        if not thinking:
            logits = logits + guide.bias()  # guide only acts after </think>

        token = sample(logits)
        ids.append(token)
        if token == eos:
            break

        if thinking:
            thought += 1
            if token == think_end:
                thinking = False
            elif thought > max_think_tokens:
                ids.append(think_end)  # out of budget, end thinking for it
                thinking = False
        else:
            guide.advance(token)

    return ids
```

I also added quantization and CUDA support so I could run these on my RTX 3080. Even so, a game took about an hour on average. I ran three models, Qwen3-4B, Qwen3-8B, and DeepSeek-R1-Distill-Qwen-7B, across the same five board formats from last time.

## The first attempt

I ran the games in batches. After the first batch, I noticed something was wrong. The model virtually never agreed with the guide at all.

Upon closer inspection, about 99% of the time it wanted to emit `\n\n` after thinking. My guide only allowed tokens that begin a legal move, so it banned the one token the model actually wanted. The first token of every move was drawn from whatever was left, and everything after it was conditioned on that. The moves were basically noise.

I realized I had missed something, but I committed to finishing the full run so I would have a complete data set.

![Thinking models, guided, baseline settings](/assets/images/considering-the-limited-time/guided-thinking-baseline.png)

First-token confidence is effectively zero everywhere, and overall confidence never gets past about 20 percent. The long games look like progress, but it's the same effect I saw in SmolLM2's output last time. It is basically just random noise filtered through the guide.

## Read the docs

I should have started here.

Qwen publishes [recommended sampling settings](https://qwen.readthedocs.io/en/latest/getting_started/quickstart.html) for thinking mode and a [guide to thinking budgets](https://github.com/QwenLM/Qwen3/blob/main/docs/source/getting_started/thinking_budget.md). When the budget runs out, you are supposed to inject a sentence into the end of its reasoning:

> Considering the limited time by the user, I have to give the solution based on the thinking directly now.

Then close thinking and follow it with `\n\n`. I added that for the Qwen3 models and switched them to the recommended sampling.

I couldn't find the equivalent docs about R1, but noticed it emitted the double new lines. I also updated it to run with the `\n\n` addition, but no special nudge sentence.

Then I reran all of the games:

![Thinking models, guided, tuned settings](/assets/images/considering-the-limited-time/guided-thinking-tuned.png)

The signal is back. First-token confidence is between about 65 and 75 percent on SAN for all three models. The games loop sooner because the models are coherent enough to get stuck in positions they have already seen.

It is still below what non-thinking Qwen2.5-7B managed last time, mostly 70 to 100 percent. That is not a clean comparison. It is a different model generation, quantized, with different sampling. But the purple line is the more interesting part of the chart.

## The right budget

Qwen's docs also recommend a thinking budget of 32k tokens. I ran with a budget of 2k because 32k would take too long. The purple mean thinking tokens line shows the harness truncated thinking much of the time, especially on the ASCII and FEN versions where the model needs to think a lot about the board format.

I expected it to have some negative effect, but not as much impact as it ended up having.

| model         | cut off     | finished    |
| ------------- | ----------- | ----------- |
| Qwen3-4B      | 28% (n=474) | 64% (n=74)  |
| Qwen3-8B      | 32% (n=296) | 100% (n=28) |
| R1-Distill-7B | 23% (n=381) | 47% (n=165) |

This is average first-token confidence grouped by whether thinking was cut off or finished.

At this point, I had put in all the time I wanted to on this. Running a full run with 32k tokens would take days. I may do it out of curiosity, but want to move on to some other topics first. To get a sneak peek, I chose a mid game ply where thinking got cut off and the model was out of alignment with the guide, then let it think as much as it wanted and compared the result.

```
# King moves to d1, escaping check

Token                              K       d       1
Probability (truncated thinking)   0.05%   25.58%  0.00%
Probability (full thinking)        100.0%  0.00%   0.00%
```

In this position, this was the only legal move. Allowed to think fully, it settled on moving the king, but not in a legal way. Its thinking trace is 12k characters of rambling circular confusion. I think this is about the limit of what an 8B model can do with this harness. To be fair to the model, this harness is designed to be challenging.

## Scope and limits

The harness is not designed to maximize chess playing ability. The purpose is just to explore the alignment issue. I only ran one game per condition. The results are illustrative, but not real measurements.

I ran everything at nf4. Quantization may impact the output compared to a full fp16 model. I did not test that aspect.

The 2,048-token budget is far below what Qwen recommends, and most turns hit it. The truncated/finished split is not controlled for position difficulty, and the full-budget replay is one position.

Run it yourself: [repo](https://github.com/jdhornsby/guided-decoding)

I am still not a good chess player.