# Claude Skills Portfolio

A collection of custom "skills" I've designed for Claude — reusable, spec'd workflows that turn a repeatable task into something Claude can execute reliably, end to end, without me re-explaining the process every time.

Each skill folder contains:
- **`SKILL.md`** — the actual spec: trigger conditions, required inputs, step-by-step execution logic, and a verification checklist. This is the artifact Claude reads and follows.
- **`CASE_STUDY.md`** — the product story behind it: the problem, the scoping decisions, what broke in practice, and how the spec evolved.

The skills themselves don't need to share a theme. What they have in common is the *process* — how each one is scoped, constrained, and hardened against real-world failure.

## Skills

| Skill | Category | Problem it solves | Status |
|---|---|---|---|
| [raw-footage-highlight-reel](./skills/raw-footage-highlight-reel) | Media / creative automation | Turns a folder of raw, unedited trip/ride footage (GoPro + phone dumps) into a curated, chaptered highlight video at one or more target lengths | Shipped |

*(More skills added here as they're built.)*

## Why this matters as a body of work

Anyone can write a one-off prompt. The harder — and more transferable — skill is:
1. **Scoping** what should be decided by the user vs. decided automatically (and being explicit about which is which).
2. **Designing for failure** — anticipating where an agentic workflow will break (timeouts, storage limits, corrupted intermediate files) and specifying recovery paths before they're needed.
3. **Writing verification into the spec**, not bolting it on after — every skill here ends with an explicit "how do we know this worked" checklist.
4. **Iterating from real runs** — each `CASE_STUDY.md` documents what actually happened the first time the skill ran, and what changed in the spec as a result.

That's the same muscle as writing a PRD or an eng spec — just applied to specifying an AI agent's behavior instead of a human team's.
