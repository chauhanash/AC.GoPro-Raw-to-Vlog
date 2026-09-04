# Case Study: Raw Footage → Highlight Reel

## The problem

Trip footage — GoPro handlebar cams, phone camera-roll dumps, mixed sources — piles up as hours of raw, unwatchable video. Turning it into something shareable is a real editing job: reviewing every clip, cutting duplicates and dead time, sequencing a narrative, matching pacing to a target length, and rendering a clean final file. It's tedious enough that most people just... don't, and the footage sits untouched.

The goal was to hand this whole job to Claude, operating directly on the user's own machine (footage never leaves the device), and have it consistently produce a finished, watchable video — without babysitting every step.

## Key scoping decisions

**Always present a plan before doing any rendering — every run, not just the first.**
This was made a hard requirement rather than a nice-to-have. Early on it would have been easy to let Claude infer preferences (source priority when cameras overlap, portrait vs. landscape handling, pacing style) and just proceed. Instead, the spec forces an explicit checkpoint: summarize the footage, propose a structure, and ask before spending any compute on rendering. The cost of guessing wrong here isn't a bad suggestion — it's minutes of wasted render time and a result the user has to throw away. That trade-off is what justified the extra step.

**No default priority between cameras.**
When footage from two devices overlaps the same moment, the spec explicitly refuses to pick a default — it's flagged as a decision for the user, every time. This was a deliberate call: a "smart default" here would occasionally be wrong in a way that's invisible until the user watches the final cut and notices the "wrong" angle was used.

**Reference-video pacing, used as a cue rather than a template.**
The skill supports an optional reference video to calibrate cut speed and energy. In practice, the reference available for the first real run was a 75-second sponsored clip — good for pacing/energy cues, too short to structurally template a 30-minute edit. The spec reflects that distinction explicitly rather than pretending the reference did more than it did.

## What broke in practice, and how the spec changed

The first real production run (a 7-day, 112.8 GB, 125-clip motorbike trip) surfaced failure modes that hadn't been designed for up front — each one became a hardened step in the spec:

- **Shell command timeouts on long batches.** The device shell caps commands at ~45 seconds, and backgrounded processes (`nohup ... &`) don't survive past the call that launched them — they get killed with the rest of that call's process tree. A multi-minute rendering job can't run as one call. **Fix:** the spec now requires a resumable script — one that skips already-completed work, self-limits to ~30 seconds, and gets re-invoked repeatedly until done. This pattern is now used for thumbnailing, segment rendering, and any long batch.

- **Stream-layout mismatches silently corrupting the final concat.** GoPro clips carry embedded timecode metadata; re-encoding without stripping it caused the mp4 muxer to silently add an extra data-track on some segments but not others (e.g. not on generated title cards). The mismatch only surfaced as a corrupted concat much later. **Fix:** `-map_metadata -1` is now a required flag on every re-encode, and every segment is validated with `ffprobe` immediately after rendering — checking duration and audio-stream presence, not just a zero exit code.

- **The delivery bottleneck.** A finished export can be multiple GB. Every available file-transfer path capped out well below that (a small-file preview tool capped at 30MB; a general commit/staging path capped at 20MB per file, 100MB per call). Writing the final file directly to the mounted/networked destination was also unreliable — a single large write could blow the 45-second command cap mid-write, leaving a truncated, unplayable file. **Fix:** the spec now mandates a three-stage delivery path: render and concatenate on fast *local* scratch space first, transfer to the destination in size-bounded chunks with matching byte offsets, then checksum both copies before calling the job done. This is now flagged as a hard constraint, not an implementation detail — the wrong assumption here silently produces an unusable file, hours into the job.

Each of these went from "unexpected failure during a real run" to "an explicit, numbered rule in the spec," so the next run doesn't rediscover the same failure the hard way.

## Outcome

- A ~7-day, 125-clip, 112.8 GB source trip was turned into two deliverables — a chaptered 30-minute cut and a 3-minute highlight montage — sharing one curation pass and one set of rendered segments.
- A second, unrelated trip (mixed GoPro/phone/flights/city footage, no motorbike, no reference video available) ran through the same skill with different preferences (landscape-only phone footage, no reference-based pacing) and produced a 5-chapter ~11-minute cut on the first pass, refined once on feedback.
- The spec is now reusable across trip types without rewriting it per trip — the parts that are genuinely trip-specific (chapter structure, pacing style, source handling) are treated as inputs to gather, not assumptions baked into the logic.

## What this demonstrates

The interesting part isn't the ffmpeg commands — it's the product judgment underneath them: which decisions to force a checkpoint for versus automate, how to design an agentic workflow so its failure modes are recoverable rather than silent, and how to fold real production failures back into the spec so they don't recur.
