---
name: raw-footage-highlight-reel
description: "Turn a folder of raw trip/ride footage (GoPro, phone, mixed sources) into a curated highlight-reel video at one or more target lengths, styled after an optional reference video. Always opens by analyzing the footage and presenting a plan (with options where choices matter) before doing any heavy rendering. Runs on the user's local machine via the device bridge, originals untouched."
---

# Raw Footage Highlight Reel

## Trigger
User has a folder of raw video (GoPro exports, phone camera-roll dumps, mixed sources) and wants it compiled into a shorter, watchable highlight video — with an explicit or implied target duration (e.g. "make this a 30 minute video", "give me a 3 minute cut too"), optionally matched to the pacing/style of a reference video they provide.

## Required inputs (ask via AskUserQuestion if not given up front; some are also settled during the plan step below)
1. **Source folder** — path on the user's device containing the raw footage. Never write into or delete from it directly; all work happens in a new subfolder created inside it (e.g. `_ClaudeEdit/`).
2. **Style reference** (optional) — a local sample video path, or a YouTube URL. Used only to calibrate pacing/shot-length feel, not to copy content. Analyze with the video-analysis tool (works on an uploaded file or YouTube link; accuracy drops on long videos, so warn the user and prefer a short reference or a representative chunk).
3. **Target duration(s)** — one or more lengths (e.g. 30 min AND 3 min). Multiple lengths can reuse the same curation pass (see Step 9).

Do not ask about source priority between cameras up front, and don't apply a fixed default priority between devices/cameras — if footage from multiple cameras genuinely overlaps the same moment, surface that as a choice in the plan step below instead of deciding it silently.

## Steps

1. **Connect and verify.** Request device folder access to the source folder. Verify on the device (via the local shell bridge) that `ffmpeg`, `ffprobe`, `python3`, and PIL/imagemagick are present. That shell has NO network access — if a tool is missing, tell the user to install it themselves; don't try to install over that connection.

2. **Build a footage manifest.** Recursively enumerate video files (mp4/mov/etc., case-insensitive). For each, probe with `ffprobe` (`-show_format -show_streams` in JSON, not a restrictive `-show_entries` string — side-data fields like rotation aren't reachable that way) for `creation_time`, `duration`, `width`/`height`, rotation/side-data (to compute true orientation), and whether an audio stream exists. Save as `manifest.csv` in the new subfolder.

3. **Present a plan before doing any heavy work — every single run, not just the first.** This is a hard requirement, not an optional courtesy. From the manifest alone (no thumbnails needed yet), summarize in plain conversation: the date/time range covered, a breakdown of footage by source/camera and how much of it is usable video, total raw duration, and any coverage gaps (days/periods with no usable footage). Propose a concrete structure (e.g. chapters by day, by location, or by theme) and a rough sense of how the target duration(s) will be reached (e.g. "55 minutes raw → a curated ~11 minute cut"). Then use AskUserQuestion to surface any real decision points before proceeding — typical ones include: orientation filtering (landscape-only vs include portrait), whether to include still photos alongside video, how to handle footage where multiple cameras overlap the same moment (there is no default priority — ask), overall pacing/style (e.g. relaxed vs fast-cut) if no reference video was given, and confirmation of the proposed chapter structure. Only move on to thumbnailing/rendering once the user has responded (or, if the session is clearly unattended, state the assumptions made and proceed).

4. **Generate thumbnails.** One mid-point frame per candidate clip (`ffmpeg -ss <midpoint> -frames:v 1 -vf scale=480:-2`), scaled down, into `thumbs/`. Time a small batch first (expect roughly 0.2-0.4s/clip for short clips) and size subsequent batches to fit the ~40s command budget (see Step 9 for why that limit exists).

5. **Build labeled contact sheets.** Grid the thumbnails (PIL), grouped by day or logical session, each cell labeled with index/source/filename/timestamp/duration. Save to `contact_sheets/`.

6. **Stage and visually review.** Stage each contact sheet into the cloud workspace (pass the DEVICE's own absolute path — e.g. the Windows path — not the local shell's internal mount path) and actually look at each one. Curate deliberately, applying whatever the user chose in Step 3:
   - Cut near-duplicate/repetitive shots of the same subject.
   - Cut low-value filler (blank/dark frames, generic transit shots, repeated establishing shots).
   - Watch for the SAME original clip appearing twice under different filenames (e.g. an iPhone Photos edited export like `IMG_1823` next to `IMG_E1823` — same underlying footage). Keep at most one.
   - Favor visually distinct, narratively meaningful moments over exhaustive coverage.

7. **Build a chaptered EDL** (JSON: chapter title cards + ordered clips with target on-screen durations). Hard pacing rules — apply these regardless of what the raw curation picks:
   - Never chain two-or-more consecutive clips that are both under ~3 seconds unless the shot is a genuine standout (a single unmistakable "money shot" can stand alone as a quick flash; two mediocre quick cuts in a row cannot).
   - Never place two segments sourced from the same original file back-to-back.
   - Keep chronological order within/across chapters unless there's a clear reason not to.
   - For long source clips, use a centered trim window rather than the full length.
   - Scale total target duration to roughly match the requested runtime (pad/trim per-clip targets proportionally, each clamped to that clip's real available length); for a second, shorter target length from the same pass, drop lower-priority clips and/or tighten per-clip durations rather than re-curating from scratch.

8. **Render each segment**, normalized to a consistent profile (e.g. 16:9 crop+scale, fixed fps like 24, short video+audio fade in/out), plus a silent-audio title-card clip per chapter so every segment shares an identical stream layout. Non-obvious but critical:
   - Keep original clip audio — never `-an`. Map explicitly (`-map 0:v:0 -map 0:a:0`) and re-encode audio to one consistent codec/rate (e.g. AAC 48kHz stereo) across all segments.
   - Add `-map_metadata -1` when re-encoding clips carrying embedded timecode (common on GoPro). Without it, the mp4 muxer silently auto-adds an extra `tmcd` data-stream track on some segments but not others (e.g. not on generated title cards), and the stream-layout mismatch corrupts the later concat.
   - After rendering, validate each segment with `ffprobe` — duration ≥ target AND an audio stream present — don't trust a zero exit code alone.

9. **Execution model for long batches.** Local-device shell commands are capped at ~45 seconds AND each command runs in its own fresh, isolated process — `nohup ... &`-style backgrounding does NOT survive past the end of the call that launched it (it gets killed with the rest of that call's process tree). Never rely on backgrounding for a multi-minute job. Instead, write a resumable script that: skips any EDL entry whose output already exists and validates correctly, tracks its own wall-clock time and stops itself around 26-33 seconds, and gets invoked repeatedly (also wrap each invocation in the shell's own `timeout 40 ...` as a second safety net) until it reports everything done. Use this same pattern for thumbnail generation, per-segment rendering, and any other batch of many short ffmpeg jobs.

10. **Final assembly.** Concat the surviving normalized segments (`ffmpeg -f concat -c copy -movflags +faststart`) into a file on FAST LOCAL scratch space inside the device shell (e.g. `/tmp/output.mp4`) — not directly onto the connected/mounted folder. Direct writes of large files to the mounted folder are slow (observed ~8-9 MB/s) and a single call moving hundreds of MB will blow the ~45s cap, leaving a truncated/corrupt file (`moov atom not found`).

11. **Transfer to the destination in chunks.** Copy from local scratch to the destination folder using `dd` with matching `skip=`/`seek=`/`count=` (in 1M blocks; measure throughput on the first chunk, typically 150-250MB fits one call), `conv=notrunc`, truncating the destination only on the very first chunk (`: > dest`). After the last chunk, `md5sum` both copies and confirm they match before telling the user it's done.

12. **Delivering the result.** A real finished trip video is usually hundreds of MB to a few GB — generally too large for the per-file/per-call staging caps into the chat, so it typically can't also be dropped in as an inline preview. Tell the user the exact file path in their folder rather than implying it's only in the conversation. Offer a smaller/lower-res proxy render only if they explicitly want an inline preview and are OK with the extra encode time.

13. **Multiple target durations from one pass.** Reuse the same manifest, contact-sheet review, and normalized per-clip segments across all requested lengths — build a second (shorter) EDL referencing the same rendered segments/subset, then repeat Steps 10-11 for that second output. Don't redo curation or thumbnailing per length.

14. **Never touch the originals.** All new files (manifest, thumbnails, contact sheets, segments, concat lists, rendered outputs) live under the `_ClaudeEdit/` subfolder inside the source folder. Don't attempt to delete anything through the device bridge (it's blocked by default). Re-rendering a segment means overwriting via ffmpeg's own `-y`, not removing the old file first.

## Verification
- `ffprobe` the final output: duration within the requested target range, both video and audio streams present, resolution/fps as intended.
- Spot-check chapter boundaries and any clip near the "too short" threshold — confirm no back-to-back sub-3s or duplicate-source cuts slipped through.
- Confirm checksums match between the local-scratch copy and the copy written into the connected folder.
- Tell the user the exact delivered path, the runtime, and a one-line recap of what was cut/prioritized, so they can give targeted feedback (pacing, chapter balance, wrong shot picked) rather than starting over.