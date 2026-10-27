# KwikScript log

## Now

Finishing KwikScript as a native SwiftUI app, free forever: a Descript replacement (edit by text, silence/filler cleanup, simple styled captions, basic audio clean-up, export video or FCPXML). The lasting output is KwikCore, the shared Swift package KwikLog, KwikCut and KwikCast build on; AI editing and in-depth captions belong to KwikCut. Main goal (2026-10-17): AI editing with our own render, no NLE; FCPXML is a secondary export, and this app's word/silence cutting is the bar KwikCore must match. Plan: `NATIVE-PLAN.md`. Licence: option A, MIT in new repos. Milestone 0 (KwikCore) done 2026-10-16 at `~/Documents/Personal Projects/KwikCore` (committed locally, no remote), plus the edit model, cleanup tools and FCPXML export via `kwik cut`; milestone 1 (all native, no Electron) comes next, features after.

## Next

- [x] P1 · milestone 0: new MIT KwikCore repo with BEHAVIOUR.md, decode, peaks, transcriber library, `kwik transcribe` CLI (brief: NATIVE-PLAN.md §7) · added 2026-10-16
- [ ] P2 · import `~/Desktop/DJI 0018 cut (kwik).fcpxml` (from `kwik cut`) in Final Cut and check relink, timecode and how tight the cuts sound · added 2026-10-16
- [ ] P2 · this app's FFmpeg WAVs keep AAC priming, so transcripts of DJI files are ~44 ms late against Final Cut (KwikCore `docs/PERFORMANCE.md`); fix here or only natively? · added 2026-10-16
- [ ] P2 · name the native app's new repo and decide whether it replaces ricardo-nth/kwikscript · added 2026-10-16
- [ ] P2 · decide on the Apple Developer Program for a notarized free release · added 2026-10-16
- [ ] P2 · pick the offline default (Parakeet v3 or Ultra) from the comparison session, and check Ultra has a Core ML build · added 2026-10-16
- [ ] P2 · test Scribe v2 keeps fillers verbatim before relying on it for cleanup (milestone 4) · added 2026-10-16
- [ ] P3 · decide whether ROADMAP.md's open items move here or it stays the public roadmap · added 2026-10-16

## Done

- 2026-10-17 · milestone 1 native app built at `../N3Studio` (MIT, no commits yet): opens, transcribes, cleans up, edits by word, plays gaplessly and exports MP4/FCPXML with no Electron; idle 120 MB against this app's ~452 MB on the same 20 min file; waiting on a hands-on run · KwikScript-native LOG.md
- 2026-10-16 · KwikCore ahead of plan: edit model, filler/pause/quiet cleanup and FCPXML 1.11 export with source timecode; `kwik cut` takes clip 0018 from 9:28.9 to 8:24.2 (25 fillers, 175 pauses) · KwikCore c88e3ea
- 2026-10-16 · milestone 0 done in KwikCore (committed locally): `kwik transcribe` output identical to the bundled helper on 3 DJI clips, AVFoundation decode 2.1 s / 30 MB against FFmpeg 0.6 s / 22 MB on 9.5 min 4K · KwikCore LOG.md, docs/PERFORMANCE.md
- 2026-10-16 · confirmed the opt-in offline proxy (540p copy in the cache, originals for render and FCPXML) · NATIVE-PLAN.md §3
- 2026-10-16 · added projects (Projects window, autosave, opt-in offline proxy) to the plan; milestone 0 started · NATIVE-PLAN.md §3, §5
- 2026-10-16 · scope revised: Descript replacement, free forever; online Scribe v2 opt-in, simple burn-in captions, basic audio clean-up, video export in milestone 1; EDIT-THIS.md superseded (moved to KwikCut/KwikLog), EditCore folded into KwikCore · NATIVE-PLAN.md
- 2026-10-16 · chose licence option A (MIT, new repos, clean-room from behaviour); third-party licences checked, all permissive · NATIVE-PLAN.md §4
- 2026-10-16 · wrote the native rebuild plan: feature inventory, KwikCore API, app architecture, licence options, milestones · NATIVE-PLAN.md
