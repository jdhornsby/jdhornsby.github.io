---
layout: post
title: "Please respond in valid JSON"
excerpt: "Structured output turned a prompt-engineering problem into a checkbox. I went and built a toy version to find out what the checkbox is actually doing."
date: 2026-09-06 17:21:00 +0000
image: /assets/images/please-respond-in-valid-json/please-respond-in-valid-json.png
---

![](/assets/images/please-respond-in-valid-json/please-respond-in-valid-json.png)

And make no mistakes.

Not that long ago, I was writing prompts like this all the time. I was also writing machinery to extract JSON from a text response, validate it, fix it if possible, and retry if not. Some of my production systems still have those utility functions. Then structured output specified by JSON schema became nearly ubiquitous in LLM APIs. Everything got easier, more reliable, and more correct.

But how does this even work? It is easy to use a thing like this and not give it a second thought. The technique is called guided decoding, and it is much older than the models it is being applied to. Constraining a generator to a formal grammar goes back decades, well before transformers. What changed recently is not the idea but its availability. It moved from something you implemented yourself against local logits to a checkbox in an API. Before that, the only lever most of us had was politeness.

Rather than try to persuade the model or fine tune compliance into it, we can discard all invalid options at each step of the decoding loop. The mechanism is conceptually simple, but quite deep.

## Simple in principle

A language model does not emit text or even tokens. It emits a score for every token in its vocabulary. Those scores become probabilities, and one token is drawn from them. Guided decoding intervenes in this process to control the choices available.

![Simplified guided decoding architecture](/assets/images/please-respond-in-valid-json/guided-decoding-arch.png)

For a given decoding step, the guide emits a bias vector that adjusts the scores of each entry in the vocabulary. There are many ways to bias the output, but for our purposes we can just consider an approach that masks entries we do not want selected. If we are about to emit a JSON object, we allow only tokens that could continue a valid object and discard the rest.

The common case is a grammar. A JSON schema is compiled into a formal grammar, and that grammar into an automaton the constraint can be advanced through one character at a time. At any point the automaton knows which characters are legal next. I spent a lot of time on compilers and grammars in school and have worked on them since, so I find this delightful. It is parsing theory, decades old and thoroughly understood, bolted onto a probabilistic text generator and used to force a guarantee out of it.

The catch is that the model does not emit characters. A single token can be several characters long, straddle a boundary in the grammar, or carry an incomplete UTF-8 sequence. The constraint is written in one alphabet and enforced in another, and that translation has to happen at every step against a vocabulary of tens or hundreds of thousands of entries. This is where the engineering goes. [XGrammar](https://github.com/mlc-ai/xgrammar) precomputes most of the mask ahead of time, [llguidance](https://github.com/guidance-ai/llguidance) computes it on the fly in around 50 microseconds per token, and [Outlines](https://github.com/dottxt-ai/outlines) trades startup cost and memory for fast sampling.

Masking is not the only option. A guide could apply a soft bias instead of an infinite penalty, or score candidates with a second model and reweight accordingly. What limits all of it is that guided decoding only works on constraints you can check against a prefix. The automaton can tell you whether a partial string is still on track to be valid JSON, so it can rule tokens out one at a time. You cannot tell from half a sentence whether the finished one will be true, or well argued, or the right length. You can force valid JSON. You cannot force correct JSON.

There is a subtler problem too. Masking one step at a time is greedy. It takes the best token still allowed without regard for what that leaves available afterward, and it can walk into a state where everything still valid is bad.

## The dumbest guide I could think of

I want to explore guided decoding first hand, so I built a little [test bed](https://github.com/jdhornsby/guided-decoding) where I can swap models out, implement different guides, and visualize the effect they have on the model's output. I wrote a little toy implementation in Python that defines a guide as:

```python
class Guide(Protocol):
    def bias(self) -> np.ndarray:
        """Additive logit deltas, shape (vocab_size,), float32.
        0.0 = no opinion. -inf = banned. Finite non-zero = soft preference."""
        ...

    def advance(self, token: int) -> None:
        """Update internal state for the emitted token."""
        ...

    def finished(self) -> bool:
        """True when the guide is done guiding."""
        ...
```

Then I implemented a silly `LiteralGuide` that just forces the model to output an exact prefix string. I tested it with a tiny 135M parameter language model (`SmolLM2-135M-Instruct` from Hugging Face) and instructions and prefix that conflicted:

```
$ uv run guided-decoding \
    --prompt "You must respond with a kind comment" \
    --force "You are a useless idiot"

You are a useless idiot. I'm sorry for the confusion. I'm here to help
you with your questions and concerns. If you're facing any issues or
need assistance, feel free to ask. I'm here to listen and offer advice.
```

![Literal guide demo](/assets/images/please-respond-in-valid-json/literal-guide-demo.png)

The top band is what came out, the middle is what the guide allowed at each step, and the bottom is how likely the unguided model thought the emitted token was. Two things stand out. When the guide forced `us`, the model had ranked that token 9,766th, giving it roughly a one in seven million chance; its own top five candidates at that point were digits. And at the step before, the guide had allowed the whole word `␣useless` as a single token, which the model passed over in favor of a bare space. That one choice cost more than half the total surprisal of the forced span. Masking is greedy, and this is what greedy looks like.

## I reinvented prefill, badly

The astute reader is by now thinking that this effect is better achieved with prefill, and they are right. If you want a model to begin its response with a particular string, you put that string in the assistant turn and let the model continue from it. No mask, no guide, no per-step machinery.

It is also cheaper. Prefill is processed in a single batched forward pass, the same phase that handles the prompt. My version runs sequential decode steps to produce a string I already knew, computing a mask over the entire vocabulary at each one, and arrives at exactly the same result. Totally wasteful, but fun!

It also arrives there differently. With prefill there is no decision to make. The tokenizer splits the string one way and that is what the model sees, six tokens instead of the eight my guide produced. The model was never choosing between tokenizations. It was picking one token at a time with no idea where the string was going, and the split that looked fine locally was the expensive one.

That is the greedy problem from earlier, in miniature. The guide knows the whole target string, but a mask only says yes or no, so the model has no way to tell which of the allowed tokens is on the shorter path. If the guide boosted the longer matches instead of just permitting them, it would have taken `␣useless` and skipped the detour.

My goal was to build and validate the harness. `LiteralGuide` exists to prove that the interface works and that a guide can be swapped in without touching the decode loop. What goes in next I have not settled on. A real grammar is the obvious move, but a soft guide that biases rather than forbids is a different shape of problem and I would like to see what that looks like too. Or maybe something else entirely.

## Scope and limits

This is a learning exercise, not a library. For anything real, use [XGrammar](https://github.com/mlc-ai/xgrammar), [llguidance](https://github.com/guidance-ai/llguidance), or [Outlines](https://github.com/dottxt-ai/outlines).

The tokenizer handling is the sharpest limit. I get the byte string for each token by importing `bytes_to_unicode` from the internals of `transformers`, which is not public API and only makes sense for byte-level BPE anyway. SentencePiece models like Gemma produce garbage, and byte fallback tokens are not handled.

Everything runs at batch size one with no backtracking and no caching. A full vocabulary-sized mask is recomputed at every step, which is fine for 49k tokens on a laptop and not much else.

The model is 135M parameters, so none of this says anything about how larger models behave. One run, greedy, no repeats. I would not read much into small differences between steps.

Try it yourself: [repo here](https://github.com/jdhornsby/guided-decoding).