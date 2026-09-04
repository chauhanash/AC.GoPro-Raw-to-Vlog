# Raw Footage → Highlight Reel

Turns 100+GB of raw trip footage into a finished highlight reel — spec'd with explicit user checkpoints, timeout/failure handling, and delivery verification built in, not bolted on.

## What's in this folder

- **[`SKILL.md`](./SKILL.md)** — the spec Claude actually executes: trigger conditions, required inputs, a 14-step process, and a verification checklist.
- **[`CASE_STUDY.md`](./CASE_STUDY.md)** — the product story: why key decisions were scoped the way they were, what broke during the first real production run, and how the spec was hardened as a result.

## Why this is here

This isn't a demo of "AI editing video." It's an example of specifying an agentic workflow the way you'd spec anything else a system needs to do unattended and correctly:

- **Decide what needs a human checkpoint vs. what can be automated** — and be explicit about which is which (e.g. camera-source priority is never defaulted; it's always surfaced as a choice).
- **Design for the failure modes, not just the happy path** — shell timeouts, silent stream-corruption on re-encode, file-size limits on delivery. Each one is a rule in the spec because it actually happened once and needed a real fix.
- **Write verification into the process itself** — the spec doesn't end at "render the file"; it ends at "confirm checksums match and tell the user exactly where it landed."

Read `CASE_STUDY.md` for the details on what actually broke and how the spec evolved.
