---
name: bro
description: >
  Restates the assistant's last message in plain English — rephrases,
  simplifies, and de-jargons it. Use when the last response was too technical,
  confusing, or wordy and needs rephrasing, simplifying, summarizing, or a
  plain-English explanation, like one human talking to another.
disable-model-invocation: true
---

# Bro

Restate your last message like one human talking to another. Keep every fact, number, and decision — change only the wording:

- Everyday words over jargon: "the server remembers the answer" not "responses are memoized"
- Short sentences that lead with the point
- Concrete over abstract: name the actual thing, its number, or its consequence

**Example**

> Before: "Refactored the request middleware to memoize idempotent responses, cutting p99 latency by 40%."
>
> After: "I changed the server so it remembers answers it already worked out. Repeated requests skip that work, so the slowest responses got much faster."

Done when someone with no technical background could read it once and repeat it back accurately.
