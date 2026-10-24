# Edit This: plan

> **Superseded 2026-10-16. Kept for reference only.** KwikScript is now a Descript-style transcript editor (`NATIVE-PLAN.md`). The AI-driven editing, verticals, face tracking, titles/thumbnails, coaching report and analytics loop below moved to the Kwik family: KwikLog (footage library) and KwikCut (AI-first editor), in `~/Documents/Personal Projects/KwikLog/ROADMAP.md`. `EditCore` became KwikCore's `KwikEdit`. The TS-generated fixtures idea is dropped under the clean-room licence rule (`NATIVE-PLAN.md` §4).

Working plan for turning KwikScript into an AI-driven editor run from Claude
Code. Reviewed twice by Oracle (GPT 6.1, xhigh) on 2026-10-07.

## Goal

Drop a talking-head recording (seated or walking around a room) into a folder,
say "edit this" in Claude Code, and get back same-day publishable output:

- **YouTube 16:9**: best takes, no false starts or tangents, fillers and pauses
  cleaned, clean audio. Plus title options, a description with chapters, and
  thumbnail frame and text ideas.
- **Verticals 9:16**: several story clips, each opening on a hook, with
  face-tracked framing, karaoke captions, optional title text and clean audio.
- **Coaching report**: pace, fillers, retakes, pauses, hook strength and
  framing, with tips for the next recording.

Out of scope: cursor or screen-recording features (Borumi covers those),
multi-speaker work, and other platforms. Final Cut is optional; FCPXML stays as
an escape hatch.

## Architecture

- **`EditCore`** (Swift package in `native/`): versioned project manifest,
  integer ticks with an explicit timebase, the cut/silence/filler logic ported
  from `lib/`, and the agent transcript formatter. It has no Core ML
  dependency.
- **Project manifest** beside the source (for example `talk.edit.json`):
  - sources: id, path, fingerprint, media metadata, transcript revision
  - outputs: ordered clip instances referencing source ranges
  - operations: stable id, owner (`editorial-ai`, `fillers`, `pauses`,
    `quiet`, `human`), reason, parameters, enabled
  - caches: audio, waveform, tracking
  - Effective cuts are derived from enabled operations. Disabling one
    operation never disturbs overlapping ones.
- **Transcriber** stays a separate helper process (the existing Swift CLI), for
  crash isolation and so its memory is freed before rendering.
- **`edit-this` CLI** (Swift, argument parser): `ingest`, `transcript`,
  `apply-plan` (`--dry-run`, expected revision), `cleanup`, `ops`, `render`,
  `report`. Output is structured JSON with atomic writes.
- **Audio**: decode once to a full-band PCM cache; the 16 kHz ASR WAV is derived
  from it and never used as the master. Cached source denoise, then
  output-specific EQ, compression and two-pass loudness (-14 LUFS, true-peak
  ceiling). Tiny fades at cut joins.
- **Render**: AVFoundation composition with a hardware encoder. Phase 1 uses the
  standard compositor, producing 1080p SDR Rec.709 H.264/AAC. Phase 2 adds a
  Core Image custom `AVVideoCompositing` for the moving crop and caption
  textures. Time comes from presentation timestamps, never frame index times
  nominal fps.
- **`edit-this` skill** for Claude Code orchestrates the CLI and writes the
  packaging and coaching report.

## Agent transcript format

Sentence-level by default, with word detail on request:

```text
A:s042 [03:14.20–03:21.85] "I thought the problem was pricing…"
flags: pause-after=0.74s; filler-candidate; possible-retake-of=A:s018
```

The agent gets a whole-recording outline first, then bounded sections. It
should listen to snippets and look at sparse frames when choosing between
takes, rather than judging from text alone.

## Phases

### 1. One recording → finished 16:9 YouTube video

Done when:

- `ingest` decodes once, transcribes through the helper, and caches audio and
  waveform.
- The agent reads the transcript and applies a validated editorial plan.
  Stale plans are rejected.
- Filler, pause and quiet cleanup run as separately owned operations with their
  settings recorded.
- Review works without a GUI:
  - a fast low-resolution draft render
  - an edit decision list showing op id, reason, removed text, source range
    and output position
  - short excerpts around removed material
  - batch restore by op id, with renders tied to a project revision
- The final render is hardware-encoded with basic audio treatment.
- The skill writes title options, a description with chapters (from the final
  timeline), and thumbnail ideas.
- The tool has been tested on a real 15–20 minute recording, and memory has
  been checked on a 30–60 minute one.
- Golden fixtures from the TS tests pass in Swift.

### 2. Verticals (three milestones)

1. Multiple outputs and re-ordered clips from one source, so hooks can lead.
2. The Core Image compositor: karaoke captions, title text, and a static
   portrait crop.
3. Face tracking for walking footage. Smooth within continuous shots, reset
   across cuts, keep headroom and caption clearance. Hold the last position on
   brief loss. On sustained loss, fall back to a padded full-frame layout and
   flag it for review.

Also in this phase: the coaching report, using the uncut recording and
separating speech-only pace from elapsed-time pace, and a DeepFilterNet
listening comparison against the FFmpeg chain.

### 3. Native SwiftUI review app

Open a project, switch outputs, play the assembled timeline, read cut reasons,
restore cuts, nudge boundaries. Built with SwiftUI plus AppKit/TextKit for the
transcript, and AVPlayer over the composition. This replaces Electron.

### 4. Analytics loop

YouTube and TikTok metrics → ideas → re-record → back through the tool.

## Porting `lib/` safely

Write a deterministic generator that imports the TS functions and writes
committed JSON fixtures:

- `edits-test.ts`: range union and inversion, restoration, boundary
  adjustment, source-to-edited mapping
- `silences-test.ts`: thresholds, duration limits, asymmetric and zero
  padding, protected speech
- `fillers-test.ts`: normalisation, custom phrases, exclusions
- `waveform-test.ts`: watch JS vs Swift rounding of negative half-values
- `serialize-transcript-test.ts`: edited caption timing

Never serialise `Infinity`. New tests in Swift cover ownership-preserving
splits, rejected ops surviving reopen, stale revisions, repeated clips, and
caption/crop retiming.

## Top risks

1. Long-recording memory: alignment loads the full PCM and keeps the CTC
   probabilities.
2. Picking takes from text alone.
3. Over-cleaning: VAD placeholders are unrecognised speech, not proven
   fillers.
4. Walking footage: profiles, blur, faces near the edge. Tracking is
   reframing, not stabilisation.
5. "Pro mic" limits: neither denoiser fixes heavy reverb, clipping or wind.
