---
layout: post
title: "Nothing illegal happened"
excerpt: "Guided decoding can force a language model to never make an illegal chess move. Whether it meant to make the move it did is a different question, and the confidence traces show where a 7B model's chess knowledge runs out."
date: 2026-09-14 20:00:00 +0000
image: /assets/images/nothing-illegal-happened/nothing-illegal-happened.png
---

I have been interested in [guided decoding](/please-respond-in-valid-json) lately. Instead of asking a language model for well-formed output and checking it afterward, you constrain the decoder so it cannot produce anything invalid. A model emits a score for every token in its vocabulary at each step, and the next token gets drawn from those scores. A guide intervenes in that by masking out tokens that violate whatever rule you care about, so the next token is always a valid continuation.

While thinking of interesting problems to apply it to, I recalled a [Dynomight post](https://dynomight.net/chess/) I read a couple years ago about how well language models can play chess. There was one line in that post about using a grammar to force models to pick from the set of legal moves. The post was focused on which models play well and the constraint was just an unimportant detail.

That detail intrigued me. However, instead of a grammar, I wondered if I could use a chess engine. This is actually not a new idea. Ma and Hu published [Logically Constrained Decoding](https://aclanthology.org/2025.mathnlp-main.11/) last year. It extends guided decoding past syntax to world models. Chess was one of their proof-of-concepts.

I wanted to build my own implementation and see what the models were doing and maybe find an opponent I can beat.

## Bot v bot

I wanted to watch a whole game. Two instances of the same model, one playing White and one playing Black. Pure language model output. Just a prompt, a guide, and a dream. This is the best game I got.

<video controls loop muted playsinline width="100%">
  <source src="/assets/videos/nothing-illegal-happened/game.mp4" type="video/mp4">
</video>

Both players are Qwen2.5-7B-Instruct, the largest model I can run on my hardware. The number under each move is the model's confidence in it. A move is a few tokens, and I take how likely the model thought each one was and average them. Multiplying them together would punish long moves for being long.

It's not bad for a 7B model. The opening is good, there are some moves approaching real tactics, and it is almost entertaining. Then it devolves into the models shuffling pieces back and forth in a loop. That turned out to be the normal outcome rather than a bad run.

## How to represent the board

I wanted to ask the model to output the next move in standard algebraic notation (SAN). That is what drives the chess engine state, so it seemed obvious. The best way to represent the current state of the game was not.

I ran all of these experiments with instruct models, so everything is represented as a chat. I tried five approaches:

1. SAN chat - The two models chat with one another, each being the assistant in their view of the conversation. They only say moves in SAN back and forth... like two grandmasters spitting chess moves.
2. ASCII board - The model gets an ASCII representation of the board state. This is not a common way of representing a chessboard, so I was not expecting much.
3. FEN - The model gets a FEN representation of the board state. This is a standard representation, but not the most common.
4. PGN (full) - The model gets the current board state as a PGN list. I expected this to be the most common representation in training data.
5. PGN (window) - The model gets the current board state as a PGN list of up to 10 moves with history compacted into a SetUp header in FEN. I wanted to see if the more compact representation did anything.

## Unguided and lost

As a baseline, I ran a game using each prompt approach on five small models (vs themselves) with no guide. Failures to respond with just the next move in SAN were retried up to five times. After that, the player forfeits the game.

Each game was capped at 60 plies. This is how far each game progressed:

| model | san_chat | ascii_board | fen_board | pgn_full | pgn_windowed |
|---|---|---|---|---|---|
| SmolLM2-135M-Instruct | 0 | 0 | 0 | 1 | 0 |
| SmolLM2-360M-Instruct | 0 | 0 | 0 | 1 | 3 |
| Qwen2.5-1.5B-Instruct | 6 | 1 | 4 | 5 | 6 |
| Qwen2.5-3B-Instruct | 4 | 1 | 3 | 0 | 0 |
| Qwen2.5-7B-Instruct | 16 | 2 | 5 | 17 | 11 |

So, not far. The models would often start responding with prose or talking about chess. A lot of them had SAN embedded in their responses, but I was not interested in trying to parse it out. This is a post about guided decoding.

## Legal moves only

Next, I built a guide. The game driver shares its chess engine instance with it, so the guide can ask the engine for the list of legal moves for the current position. That list is the constraint. A grammar would only be able to check that the model produced something shaped like a move, but the engine knows which moves are actually available.

Turning that list into a mask is tricky. The engine gives me moves as strings, but the model emits tokens. To reconcile the two, the guide walks the vocabulary trie I built in my previous [post](/please-respond-in-valid-json) to find every token that could begin or continue one of the legal moves given what the model has emitted so far. Everything else gets masked out. Once the move is complete the guide forces the model to emit the stop token, which ends its turn.

I reran the same five models across the same five prompt styles with the guide turned on. Nothing else changed. No game ended in a forfeit this time, because a forfeit is no longer possible. Every game either hit the 60 ply cap or ended in a draw by repetition.

![Guided results](/assets/images/nothing-illegal-happened/guided.png)

The bars are how long each game went before the model started looping. The lines are how confident the model was during that stretch. Confidence is measured over the same window as the bar. The models tended to be very confident in their looping, so I wanted to exclude those portions.

The gap between the small and large models is noticeable. Qwen2.5-7B-Instruct stays between roughly 70 and 100 percent on most prompt styles, which means the guide was mostly agreeing with a move the model was going to make anyway. The two Smol models tended to not agree with the guide at all, despite some long games. The SAN prompt was the exception, but resulted in immediate looping.

A loop requires arriving back at a position the game has already been in. That requires some amount of coherence. A near random walk through legal moves almost never does it. So SmolLM2-360M-Instruct running the full 60 plies on ASCII and FEN without looping is actually it being totally confused by the board representations. Qwen2.5-7B-Instruct also runs to 60 on a couple of prompts, and it meant to.

## What the model meant

A guided model always emits a legal move by construction. The question is whether it meant to or was just random noise filtered through the guide. That is what my harness is built to find out. I record the raw distribution the model output in every decoding phase and measure how far off the guide's selections were. That is what the confidence above is based on.

The small models were not trying to output chess moves at all for the most part. The bigger models often were. Here is a capture from a 7B game, with the probability the model assigned to each token it emitted:

```
# Queen takes h7 and puts opponent's king in check

Token           Q       x       h       7       +
Probability     33.8%   49.0%   77.5%   0.0%    8.9%
```

The model was fine with moving the queen, fine with it being a capture, and getting surer about the h file. Then it fell off a cliff on the rank. The one token that requires knowing where the pieces actually are is the one it could not produce. I saw that pattern quite a few times in 7B games I ran.

If I were running this in production, I would want these numbers. Valid output is not evidence that the model is doing anything, but it might be the only thing you get to see.

## Scope and limits

This was a fun weekend project. Everything here was n=1 per condition. I probed a bit at n=4 while choosing formats, but the numbers reported are single games with fixed seeds. Different seeds produce different games, so treat the individual results as illustrations rather than measurements.

All of the models are instruct-tuned and none were run with thinking enabled. Base models might behave differently on the PGN formats and I did not test them. Thinking models are the obvious next thing to try.

Run it yourself: [repo](https://github.com/jdhornsby/guided-decoding)

I am not a good chess player.
