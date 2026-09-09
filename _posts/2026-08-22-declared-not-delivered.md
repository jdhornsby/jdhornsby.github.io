---
layout: post
title: "Declared, not delivered"
excerpt: "A planned replication of the schema-ordering experiment on a small non-reasoning model. It turned into an investigation of why Vertex reorders schema fields alphabetically before the model ever sees them."
date: 2026-08-22 19:34:12 +0000
image: /assets/images/declared-not-delivered/declared-not-delivered.png
---

![](/assets/images/declared-not-delivered/declared-not-delivered.png)

Field order sometimes changes output quality. That was the finding in the [first post](/autoregressive-schemas/) in this series, and the follow-up I said I would run was whether the effect holds on a small model that doesn't think at all, from a different family than Gemini.

I picked Qwen3-Next-80B-A3B-Instruct, served through Vertex AI's Model-as-a-Service endpoint. It ships as two separate checkpoints, Instruct and Thinking, rather than as one model with a switch, and the Instruct build doesn't emit thinking traces at all, which makes it a cleaner baseline than a reasoning model dialed to zero. Six schema variants, the same six from the first post, 72 cases each, judged for reachability the same way as [last time](/thinking-your-way-out/). The run completed and the judging finished without anything looking obviously wrong.

## What came back

| Variant | Break rate |
|---|---:|
| flat_alpha | 32.4% |
| grouped_by_type | 16.7% |
| append_order | 19.6% |
| ui_contract | 23.3% |
| alpha_nested | 17.4% |
| nested_narrative | 19.1% |

Five of the six variants land between 16.7% and 23.3%, close enough together that at this sample size I wouldn't read the ordering among them as meaning anything. The decision-ordered schema, nested_narrative, sat at or near the floor in every column of the Gemini work and comes out mid-pack here. The only variant that clearly separates is flat_alpha, and it separates by being the worst, which is not where it landed on Gemini either.

A result where five different treatments produce roughly the same number is either a real finding about the model or a sign that the treatments weren't different, and the second is much easier to check than the first, so I went and read the raw output instead of the summary.

## Where the order went

The nested variants were all coming back with their keys in alphabetical order, whatever the schema had declared. That also explained flat_alpha, which is the one variant whose declared order is already alphabetical, so it was the only case where what I asked for and what came back agreed.

If the schema was being alphabetized somewhere between my process and the model, then five of my six conditions were the same condition and the experiment had never run. The question was where the sorting happened, because that determines whether it's mine.

**Pydantic.** The schemas are Pydantic models and the request carries `model_json_schema()`. I had been using Pydantic the same way against Gemini for months without seeing this, so I doubted it was the culprit, but it's two lines to check and it's the layer closest to me:

```python
>>> list(NarrativeLevel.model_json_schema()['properties'])
['theme', 'setting', 'player_weapon', 'player_weapon_damage',
 'player_health', 'encounters', 'boss', 'objective', 'reward']

>>> list(AlphaNestedLevel.model_json_schema()['properties'])
['boss', 'encounters', 'objective', 'player_health', 'player_weapon',
 'player_weapon_damage', 'reward', 'setting', 'theme']
```

Both come out in declared order, one narrative and one alphabetical, which is what the variants are supposed to be. Whatever is sorting them is downstream of this.

**My own results files.** This is the part that had kept me from noticing earlier. The harness writes results with `model_dump_json()`, which serializes the parsed object using the field order declared on the Pydantic class rather than the order the response arrived in. The model returned alphabetical, my client parsed it into an object where order is a property of the class, and the file on disk came out in the order I expected to see. Nothing short of reading `resp.choices[0].message.content`, the response string before anything touches it, shows the ordering the model actually produced.

**The wire.** Two variants with genuinely different declared orders, run on the same case:

| Variant | Declared order | Returned order |
|---|---|---|
| alpha_nested | boss, encounters, objective, player_health, … | boss, encounters, objective, player_health, … |
| nested_narrative | theme, setting, player_weapon, player_weapon_damage, … | boss, encounters, objective, player_health, … |

Two different declarations, one output ordering. The same held at every nesting level, inside each encounter object and inside each encounter's enemy object.

**Properties order against required order.** A JSON Schema carries field order in two places, the `properties` object and the `required` array, and it seemed worth ruling out that one survived while the other didn't. I built a small schema where the properties order, the required order, and alphabetical order are three distinguishable sequences, at two levels of nesting:

| Level | properties | required | alphabetical | returned |
|---|---|---|---|---|
| top | zeta, alpha, child | child, zeta, alpha | alpha, child, zeta | alpha, child, zeta |
| nested | yankee, bravo, charlie | charlie, yankee, bravo | bravo, charlie, yankee | bravo, charlie, yankee |

Alphabetical at both levels, matching neither declared sequence.

**The SDK.** Everything up to this point had gone through the `openai` package and Pydantic, so the last thing to rule out was my client stack in its entirety. I wrote the request body by hand into a file, sent it with curl, and read the response without parsing it:

```bash
curl -s -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  --data @request.json \
  "https://aiplatform.googleapis.com/v1/projects/${GCP_PROJECT}/locations/global/endpoints/openapi/chat/completions"
```

The model's output, pulled out of the response and indented for reading:

```json
{
  "alpha_field": "zorblax",
  "child_object": {
    "bravo_field": "flimble",
    "charlie_field": "quixnar",
    "yankee_field": "snorgle"
  },
  "zeta_field": "wibblet"
}
```

Same alphabetical ordering at both levels, from a request with no Python, no SDK and no Pydantic anywhere in it.

I spent a while after this looking for a way to ask for a specific order and didn't find one. Gemini's API has a `propertyOrdering` field that does exactly this, and I tried sending it, but it isn't documented for this endpoint and I couldn't get it to have any effect, so I can't tell whether it's unsupported or whether I had the shape wrong. I also tried reaching a MaaS model through Vertex's native endpoint rather than the OpenAI-compatible one, and got back a 200 and a response that ignored my schema and invented five demonstration fields, which I take to mean that path isn't meant for these models. Neither of those is a result, and I'd treat both as me not finding the door rather than evidence there isn't one.

**The model.** Google's documentation for MaaS structured output has a usage example, a calendar event with three fields, so I ran it unmodified except for the model ID:

```python
class CalendarEvent(BaseModel):
    name: str
    date: str
    participants: list[str]

completion = client.beta.chat.completions.parse(
    model="qwen/qwen3-next-80b-a3b-instruct-maas",
    messages=[
        {"role": "system", "content": "Extract the event information."},
        {"role": "user", "content": "Alice and Bob are going to a science fair on Friday."},
    ],
    response_format=CalendarEvent,
)
```

```json
{
  "date": "Friday",
  "name": "science fair",
  "participants": ["Alice", "Bob"]
}
```

The schema declares `name, date, participants` and the response comes back `date, name, participants`, which is alphabetical. The documentation page shows the output of this example in declared order. Running the same thing against DeepSeek rather than Qwen gives the same reordering, so it isn't specific to the model.

## What this does to the experiment

Five of the six variants were never different treatments, so whatever those break rates measure, it isn't field order. The one number that survives as a real difference is flat_alpha's 32.4%, and that comparison is confounded too, because flat_alpha is a flat schema with individually numbered fields and the other five are nested schemas that all arrived alphabetized, so what separates it might be the flattening rather than the ordering.

The replication question is still open. I don't have an answer on whether the effect holds on a small non-reasoning model, and this run doesn't get me closer to one.

The earlier Gemini results are unaffected by this, and I checked rather than assuming. Gemini's native `response_schema` path returns fields in the order the schema declares, with alpha_nested arriving alphabetical because that is how it's written and nested_narrative arriving in decision order because that is how it's written. The independent variable did reach the model in those experiments.

All of this ran between 2026-07-31 and 2026-08-10, which is worth stating because this is the kind of behavior that gets changed without an announcement.

## Why this matters

In March 2024 someone opened [an issue against vLLM](https://github.com/vllm-project/vllm/issues/3283) titled "Order of keys for guided JSON." Different model, different serving stack, two years before any of this. They reported that the keys came back alphabetical rather than in the order their template declared, and that it had a significant impact on generation quality for their task. They filed it as a defect because it was breaking their work.

That is somebody I have never met, working on something I know nothing about, arriving at the premise of this series independently and reporting it as damage. I find that more convincing than my own numbers.

The same behavior has been reported against Google's own OpenAI-compatible endpoint. In June 2025 a user [showed](https://discuss.ai.google.dev/t/structured-outputs-propertyordering-field-not-respected-when-using-the-openai-compatible-api-gemini-2-flash/86790) that gemini-2.0-flash honored `propertyOrdering` through the native client in five runs out of five and failed to honor it through the OpenAI-compatible endpoint in three runs out of five, with the field confirmed present in the outgoing request. A Google engineer replied that they had reproduced it internally and would discuss it. The user asked for an update four months later, and the thread ends there. Four related threads about key ordering sit underneath it, going back to November 2024.

There is a reason this particular property is the one that keeps getting lost. Schema enforcement gets defended because a violation is loud: a missing required field or a wrong type breaks the parser on the other end immediately, so it gets caught in testing and it stays fixed. An ordering that gets discarded produces valid JSON with every field present and every type correct, nothing raises, and the only symptom is output that is a bit worse in ways nobody is going to trace back to serialization. Properties that fail quietly don't accumulate tests, and anything in the request path can drop them without a single alarm going off.

So the advice from the first post needs an asterisk that has nothing to do with models or tasks. You can order a schema carefully, ship it, and have the ordering discarded in transit, and everything you can see afterward will look right. In my case my own client library reassembled the response into the order I had asked for, so even the results on disk agreed with me. The only reason I found out is that I went and read bytes I had no particular reason to read.

## Scope and limits

One provider, one API surface, two models, tested over ten days. Gemini's native path preserves declared order and I verified that directly rather than assuming it. I could not find a documented way to control field order through Vertex's OpenAI-compatible endpoint, which is not the same as there not being one, and if somebody knows the parameter I missed I would like to hear about it. I have no visibility into what runs behind that endpoint, so this is a report of the behavior and not a diagnosis of the cause. The Qwen run and the harness changes are committed, but those numbers are an artifact of the investigation rather than data about Qwen. The reproduction is one request body and one curl command, in the [repo](https://github.com/jdhornsby/autoregressive-schemas).