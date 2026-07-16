---
title: "Evals Are What Let Me Flip the Flag"
date: 2026-07-13T08:01:00+01:00
draft: false
type: "post"
tags:
  - ai
  - evals
  - llm
  - testing
  - infrastructure
---

There's a specific kind of dread that shows up right before you change one line
of config in production. In my case the line was a feature flag that decides
which provider actually runs our LLM calls - the ones that read real customer
documents and make decisions about them. Migrating to the new provider was the
easy part. Weeks of code, sure, but code you can review. The scary part was the
moment I'd set that flag to `true` and real traffic would start flowing through
a model I hadn't watched behave on real inputs yet.

Flipping that flag felt like a gut call. My whole job for the last week was to
turn it into something boring instead.

Here's the thing about LLM features: the usual definition of "it works" doesn't
apply. There's no green checkmark that means correct. The same prompt can return
slightly different structure two calls in a row. A model swap that looks
identical in a demo can quietly get 8% worse on the long tail of weird documents
your customers actually upload. You cannot eyeball your way to confidence here.
You need evals and not just any evals, but a ladder of them where each rung
buys you a different kind of nerve.

## Rung one: fixtures

The bottom rung is the cheap one. Synthetic and anonymised fixtures, run on every
change, checking that the output has the right *shape* - the fields are present,
the types are right, the structured output parses. This is your smoke test. It
catches the dumb regressions fast and it's cheap enough to run constantly.

Fixtures don't tell you whether the model is actually *right*. A
response can be perfectly shaped and completely wrong. Fixtures keep you honest
about plumbing, but not about judgment. If you stop here (and a lot of teams do),
you've built a test suite that passes confidently while the product degrades.

## Rung two: golden datasets from real documents

The rung that actually matters is the golden set: real documents, with known-good
expected outputs, curated once and treated as sacred. This is where "does the
model get it right" lives, because these are the messy, oversized, badly-scanned,
wild inputs that synthetic fixtures never capture.

Building this set was most of the unglamorous work of the week. Real documents
are big, some were too large to run through the pipeline until I re-encoded and
compressed them. One PDF simply wouldn't compress, so it went in as a smaller
image instead. Filenames drifted out of sync with the manifest that indexed
them. None of this is intellectually hard; all of it is the tax you pay for
testing against reality instead of a tidy fiction. Pay it anyway. A golden set of real documents told me more than a thousand synthetic ones.

You want to also keep the golden set somewhere durable and access-controlled,
separate from your app data. These are the fixtures your entire confidence rests
on, and you do not want them to be casual.

## Rung three: parity runs

This is the rung that unlocked the flag, and it's the one I'd tell you not to
skip. A parity run holds the *inputs* constant and varies only the thing you're
migrating. Same documents, same expectations, old provider vs. new provider, side
by side. You're not asking "is the new provider good?" in the abstract, you're
asking the only question that matters for a migration: **is it at least as good
as the thing it's replacing, on my data?**

The beautiful thing about a parity run is that it converts a philosophical
argument ("is this model good enough?") into a diff. Either the new transport
matches the old one on the golden set or it doesn't, and if it doesn't, the
failures point at exactly which documents to go look at. When I ran the pair -
direct vs. the managed provider, on the real docs, a green result was the strongest ship signal I had, and it was cheap to
produce once the harness existed.

## What the ladder buys you

Each rung answers a different question:

- **Fixtures:** did I break the plumbing? (fast, constant)
- **Golden set:** is the output actually correct on real inputs? (slower, sacred)
- **Parity:** is the new thing at least as good as the old thing? (the migration gate)

You climb the ladder in order of cost and specificity. And the payoff is safety and *permission*. With the parity run green, flipping the flag stopped
being a decision I had to be brave about. It became a decision the evidence had
already made for me. I set it, watched the logs, and nothing happened, which is
exactly what you want a big change to feel like. boring :)

## Takeaway

If you're shipping anything backed by an LLM, the temptation is to test it the way
you demo it, a few happy-path inputs, looks great, ship. That's a trap. The
model doesn't fail on your happy path; it fails on the long tail you never looked
at, quietly, in a perfectly-shaped response.

Build the harness, build the ladder. Fixtures to catch the dumb stuff constantly. A golden set of real, ugly inputs to measure actual correctness. And when you migrate anything (provider, model, prompt strategy), a parity run so the decision to ship is a diff and not a gut feeling. Evals aren't the thing that slows you down before a big
change. They're the thing that lets you make the change and feel bored about it.
And boring, in production, is the whole point.
