---
layout: post
title: "Fingerprints of Jev"
excerpt: "Someone on Hacker News suggested fingerprinting Jev's tokenizer. The API returns how many input tokens it billed you for, so I searched for strings that separate known tokenizers and compared their counts to Jev's."
date: 2026-09-25 20:00:00 +0000
image: /assets/images/fingerprints-of-jev/fingerprints-of-jev.png
---

![](/assets/images/fingerprints-of-jev/fingerprints-of-jev.png)

A few days ago I was talking with someone on Hacker News about Jev and they suggested fingerprinting its tokenizer. The thought had occurred to me as I had recently read a few papers on that sort of black box testing ([1](https://arxiv.org/pdf/2608.29930), [2](https://isimplifyme.com/whitepapers/the-tokenizer-is-a-fingerprint), [3](https://arxiv.org/pdf/2608.31142)). Once asked, I couldn't leave it unanswered.

The general idea is simple. The API response tells you how many input tokens you were billed for. By sending a variety of strings and checking the count, you can learn about its tokenizer. You can tokenize the same strings locally with a variety of known tokenizers and compare their counts to Jev's. This won't tell you what model the API is running, but it might tell you about its tokenizer.

The hard part is choosing the strings. More on that below. First, the results:

![](/assets/images/fingerprints-of-jev/heatmap.png)

I started with a mix of common LLM and encoder tokenizers:

| Name | Source | Notes |
|---|---|---|
| cl100k | `cl100k_base` | OpenAI, GPT-4 era |
| o200k | `o200k_base` | OpenAI, GPT-4o era, also used by gpt-oss |
| Qwen3 | `Qwen/Qwen3-8B` | Alibaba |
| Llama3 | `NousResearch/Meta-Llama-3.1-8B` | Meta |
| Llama2 | `NousResearch/Llama-2-7b-hf` | Meta, SentencePiece |
| Mistral | `mistralai/Mistral-7B-v0.1` | Mistral, SentencePiece |
| Gemma2 | `unsloth/gemma-2-9b` | Google |
| DeepSeek | `deepseek-ai/DeepSeek-V3` | DeepSeek |
| BERT | `bert-base-cased` | encoder |
| RoBERTa | `roberta-base` | encoder |
| DeBERTa3 | `microsoft/deberta-v3-base` | encoder |
| XLM-R | `FacebookAI/xlm-roberta-base` | multilingual encoder |
| ModernBERT | `answerdotai/ModernBERT-base` | encoder |
| o200k-qwenpre | Frankenstein | o200k with a Qwen3 pretokenizer |

o200k was close on almost everything except digits, where Jev matched Qwen3 and Gemma2 instead. Both split numbers into single digits. I wondered how well o200k would do with Qwen's pretokenizer bolted on, so I added it to the list. It's the best match. Digits and contractions line up, and everything else is unchanged from base o200k. The remaining differences are mostly emoji, where Jev still uses more tokens than o200k.

Every difference seems to be Jev splitting text into smaller tokens than o200k does, but still within o200k's existing vocabulary. Maybe this improves numerical accuracy or something else is going on. Hard to say from the outside.

So what is Jev? My guess is something from the gpt-oss family. Its tokenizer is o200k-like. TypeSafe's CEO Diogo Almeida came from OpenAI. On [Latent Space](https://www.latent.space/p/jev) he said he wouldn't pre-train a model even if you gave him a billion dollars, and he talked about Frankensteining models together as a way to solve problems. So maybe gpt-oss was a natural starting point. Frankensteining would also fit a Qwen pretokenizer bolted onto an o200k base.

## A greedy search

The three papers I read all pick their test strings by hand. The whitepaper used 95 strings across multiple languages, code, and Unicode edge cases chosen because vocabularies tend to handle them differently. Chen's paper used 30 fixed texts, but notes they weren't sampled from a defined population. Xi's paper found that short strings are poor discriminators because different tokenizers often produce the same count on a short string by coincidence.

I tried hand-picked strings first. They gave me some initial insight, but they did not clearly separate all of the tokenizers I was testing. I wanted a sample set that guaranteed good separation between the tokenizers I was measuring against.

So I flipped it around. Instead of choosing strings and hoping they separate the candidates, search for strings that do.

I wrote a simple greedy search. A seeded random number generator makes strings of different kinds (ASCII, CJK, digits, emoji, whitespace, contractions, and so on), and each string gets tokenized by every candidate.

The goal is to build a sample that discriminates well between all of the candidates by as large a margin as I can find. Each new random string is kept only if it improves the set based on these rules:

1. **The weakest pair** Does it raise the smallest margin across all pairs?
2. **Coverage** If not, does it separate a pair within a kind of string where that pair wasn't separated yet?
3. **Anything else** If neither, does it raise any pair's margin at all?

A string that does better on an earlier question wins, and later questions only break ties.

It stops at 200 strings. The result: every one of the 91 pairs is separated by at least 21 tokens on some string:

| candidate | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 |
|---|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|
| 0 cl100k | – | 66 | 167 | 67 | 189 | 188 | 127 | 87 | 376 | 125 | 443 | 426 | 43 | 85 |
| 1 o200k | 66 | – | 109 | 59 | 202 | 196 | 137 | 38 | 334 | 125 | 426 | 412 | 73 | 85 |
| 2 Qwen3 | 167 | 109 | – | 164 | 305 | 289 | 124 | 104 | 251 | 169 | 348 | 332 | 179 | 109 |
| 3 Llama3 | 67 | 59 | 164 | – | 190 | 189 | 126 | 66 | 375 | 125 | 426 | 410 | 65 | 85 |
| 4 Llama2 | 189 | 202 | 305 | 190 | – | 21 | 265 | 209 | 512 | 146 | 485 | 469 | 155 | 202 |
| 5 Mistral | 188 | 196 | 289 | 189 | 21 | – | 259 | 200 | 509 | 137 | 479 | 463 | 151 | 196 |
| 6 Gemma2 | 127 | 137 | 124 | 126 | 265 | 259 | – | 109 | 344 | 129 | 441 | 431 | 139 | 135 |
| 7 DeepSeek | 87 | 38 | 104 | 66 | 209 | 200 | 109 | – | 327 | 133 | 419 | 407 | 88 | 85 |
| 8 BERT | 376 | 334 | 251 | 375 | 512 | 509 | 344 | 327 | – | 377 | 108 | 95 | 383 | 334 |
| 9 RoBERTa | 125 | 125 | 169 | 125 | 146 | 137 | 129 | 133 | 377 | – | 455 | 440 | 121 | 125 |
| 10 DeBERTa3 | 443 | 426 | 348 | 426 | 485 | 479 | 441 | 419 | 108 | 455 | – | 51 | 448 | 426 |
| 11 XLM-R | 426 | 412 | 332 | 410 | 469 | 463 | 431 | 407 | 95 | 440 | 51 | – | 431 | 412 |
| 12 ModernBERT | 43 | 73 | 179 | 65 | 155 | 151 | 139 | 88 | 383 | 121 | 448 | 431 | – | 77 |
| 13 o200k-qwenpre | 85 | 85 | 109 | 85 | 202 | 196 | 135 | 85 | 334 | 125 | 426 | 412 | 77 | – |

Each cell is the margin for that pair: the largest token-count gap any string in the sample set produces between the two tokenizers.

Pretty neat, if I say so myself.

## Controlling for hidden variables

A token count from an API isn't purely measuring your input string. A lot of things can change the number. The papers each ran into different examples. Here's what I controlled for, where the idea comes from, and what I did.

| Issue | Description | Source | Solution |
|---|---|---|---|
| Hidden template | The count could include a chat template, system prompt, or some wrapper | All three papers | Measure a fixed baseline and subtract it |
| Boundary merges | Characters at the edges of your string can merge with the template around it | Whitepaper, Xi | Wrap each probe in the same fixed delimiters |
| Unstable counts | The same request can return different counts (caching, serving changes) | Xi, Chen | Send every probe more than once and drop any probe whose count changes |
| Short-string coincidences | Different tokenizers often return the same count for short strings | Xi | Require the weakest-pair margin on long strings |
| Serving-stack preprocessing | The service might escape, trim, or normalize text before tokenizing it | Whitepaper (escaping) | Tested JSON escaping and Unicode normalization directly; neither explained the differences |
| Missing measurements | A rate limit or missing usage field shouldn't count as a mismatch | Chen | This didn't happen to me, but I'd drop and retry |

I also had a broken DeepSeek tokenizer at first that was producing empty sets for a lot of kinds of strings. It stood out in my sample stats due to having the same huge margin. I swapped it out with one that worked as expected. Always check your dependencies, I guess.

## Scope and limits

This fun experiment tells us a little about Jev's tokenizer. It doesn't tell us about the model behind it. My speculation about gpt-oss is just that.

I only tried some common tokenizers. There could be one out there that is a perfect match. If you know one, try it out. Repo link is below.

I am trusting that the input token count they return is accurate. They could be undercounting, rounding, or doing something else.

The test was run in late September 2026 on jev 1.13. It could change.

The repo is [here](https://github.com/jdhornsby/typesafe-jev). It contains cached counts for hashed strings so it can be run without an API key.
