---
layout: post
title: "Working as intended"
excerpt: "QA is being cut on the theory that AI can test now. AI does finish the execution half of the job, but the adversarial judgment that catches failures no spec anticipated is a different job, and a rational org cuts it anyway."
date: 2026-06-02 01:54:32 +0000
image: /assets/images/working-as-intended/working-as-intended.png
---

![](/assets/images/working-as-intended/working-as-intended.png)

QA is being cut on the theory that AI can test now. The theory is mostly right. But the role bundles two jobs under one title, and the cut can't tell them apart. One of them AI just finished. The other just got more valuable.

The first job is execution. Running cases, checking paths, regression suites, filing bugs. This was being automated for a decade before the current models, and the current models finish it. Cutting it is fair. A machine does it as well or better, and I would not argue otherwise.

The second job is a stance toward the product: probing how it will actually be used rather than how it is supposed to be, and finding where that breaks. Those are different questions, and the gap between them is where the value lives.

## The field that worked

I shipped a text input that had to be uppercase. The spec said uppercase. My validation enforced uppercase. Every case passed. Our QA flagged it anyway: users will type lowercase out of habit and get bounced by a field that is doing exactly what it was told. I pushed back. It's working as intended, nothing's broken.

She was right and I was wrong, and it took me a while to see why. Nothing was broken. Nothing was even unconventional, which is why neither I nor a model reading the spec would flag it. She knew, from watching this specific population in other software, that people type how they type, and that being blocked by a field behaving correctly does not read to a user as their own mistake. It reads as the product being broken, and they leave. The field was perfect, which is why it never read as a correctness problem at all. What broke was the fit between a correct field and how these particular people behave, and that is visible only to someone holding that context. The fix was trivial once seen: accept lowercase, uppercase it ourselves, ship. But someone had to see it coming.

The field is the easy version, because it is one rule on one screen. The stance matters more where no single screen is wrong. Picture two features in the same release. One logs you out after ten minutes idle. The other is a multi-step form that saves only on final submit. Each is correct on its own, and each passes. The problem only exists for a population that gets interrupted constantly: they fill three steps, get pulled away, and come back to a login screen and an empty form. Nothing is broken. Neither feature violates anything. The failure lives in the interaction between two correct things and a real fact about how these particular users spend their day, and it is invisible to anyone reasoning about either feature alone.

That is the shape of the work. Every piece is right and the experience is still wrong, and the job is to go find where.

## The adversary

This gets mistaken for product's job, or design's. It is a different stance. Product and design are generative. They reason about what the thing should be and how a user ought to move through it, and by role and incentive they advocate for that vision. The adversary asks the opposite question: how it will be used rather than how it should be, and where that fails.

A good one is faintly annoying, and the annoyance is the role working correctly. They don't just tell you the build is wrong, they tell you the requirement was wrong, that the thing everyone agreed to ship should not ship as written. That is more irritating than a bug report and worth more than one, and you learn to love it. Fold quality into "product owns it" and you keep the advocate and lose the adversary, because nobody paid to make the vision succeed is positioned to attack it.

## Why AI does not fill the gap

This is the part I expected to go the other way, because I spend my days pointing models at structured work and they are good at it. They are not weak here. Two separate things stop them, and I don't think either is a capability gap a better model closes, though I hold that loosely with the models moving the way they are.

The first is aim. A model will answer any question you ask about your users. It will not raise the one you did not think to ask. You can make it adversarial, tell it to attack, play the confused user, hunt for the collision between two features, but only if you already know where to point it. The aiming is the scarce thing, and the aiming is exactly the judgment being cut.

The second is incentive. Like the builder, a model is an advocate. It executes the intent you give it and optimizes the goal you set, and it is not positioned to attack that goal. Point it at a known failure mode and it will probe that mode at a scale no team can match. The model is a multiplier on the person who already knows where users fail, who can now run those modes at scale. Cut the judgment to save headcount and you keep the multiplier and lose the thing it multiplies.

## Same on paper

Here is the hard part. A good QA and a useless one look the same on paper. Same tickets, same cases run, same bugs closed. The difference shows up only when you work with them: the good one keeps finding things you would never have considered, and can tell you why each one matters, for whom and under what real condition. That is legible to a peer in an afternoon and invisible to a spreadsheet. It does not compress into a metric, so it does not travel up an org chart.

The obvious objection is that this is exactly what someone with no value would say: my contribution does not show up in the numbers, so trust me, it is there. Fair, and usually correct. But it is testable. The person with nothing makes claims that are vague and after the fact, because vague and after the fact cannot be caught being wrong. The real adversary makes the opposite kind of claim: specific, about behavior you can watch, made in advance, against pushback. **Users will type lowercase and leave** is falsifiable the second it is said. The engineer who disagreed could have been right. I was that engineer once, and I wasn't. What sets the valuable one apart is that the calls can be checked, and they're right at a rate that isn't luck. That's the part of this I can least prove on paper, which is sort of the whole problem.

## The evidence erases itself

But look at what that test needs. To know someone is right at a rate that isn't luck, you have to accumulate the calls. And the calls don't accumulate.

They erase themselves twice. Once because nothing breaks: a prevented failure leaves no ticket, so the best work produces no artifact at all. And again because when the adversary is right, the team concedes on the spot. Oh, you're right, change it, ship. Within minutes the catch is relabeled obvious, of course we handle lowercase, and the fact that one specific person saw it when nobody else did has dissolved into common sense. There is no incident to point back to, no record it was ever in question. The value was real, and it is gone from the account.

So the rate is computable in exactly one place: the head of a peer who was in the room, watching the calls get made and conceded before they evaporated. That is why the value is legible to a colleague in an afternoon and invisible to the file. I mean that mechanically. The afternoon is the only window in which the evidence exists, and the file is built after the window closes.

## Why a rational org cuts them anyway

This is why competent organizations cut these people. The local manager often knows exactly who they are, so it isn't blindness. The cut is made one level up, on the only signal that scales: headcount, tickets, "AI reduced testing effort by sixty percent." The distinction that matters was never in that signal. And it is worse than ordinary invisibility, because the best adversaries are the hardest to see in the record. Their best work is the failure that never happened, and a failure that never happened leaves nothing behind. The track record undercounts exactly the people most worth keeping.

You can't fix that by tracking harder, because the thing you would track is destroyed at the moment of concession. You can't trust the metric, because it is silent on the only thing that matters. You can't trust the artifacts, because the best work is locally testable and globally invisible. And you can't trust the local manager either, because the incentive runs the wrong way. A smaller team reporting the same tickets is a win on the manager's own scorecard, and defending an adversary means defending a headcount with a story, against a number. The one person positioned to know is the one person paid not to say.

## A sorting problem

The implication is not "don't cut QA." Much of it should go. The execution half is genuinely done, and AI finishes it. But this is a sorting problem dressed as a budget cut, and the sort can't be done from a distance. The failure is cutting while you believe the number in front of you tells you what you are giving up. Whether you ultimately cut or keep is downstream of that. An organization that knows it's blind here will at least hesitate. One that reads headcount and tickets as the whole picture will cut clean, report a tidy efficiency gain, and never learn what it lost.  
I was the engineer who said it was to spec. I was right that nothing was broken. That was never the question.
