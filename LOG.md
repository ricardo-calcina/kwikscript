# KwikScript log

## Now

Keep the Electron fork and installed transcriber working for the current local
editing and KwikLog gym workflow. Native app development is owned by
[KwikScript-native](../KwikScript-native/LOG.md); shared media, transcription,
editing and rendering work is owned by [KwikCore](../KwikCore/LOG.md).
[Kwik project map](../KwikLog/README.md#kwik-projects).

Development dependencies and vendor copies were moved to Trash on 2026-10-22.
Restore them with the README workflow before development; the installed
app/helper remain in place, and the old `dist` installers are on the SSD
(`Archive/KwikScript-electron-dist`, 2026-10-27).

Since 2026-10-27 this app gets no new features: it is Rescript-based, so it
can't be sold. KwikScript-native becomes KwikScript; this stays a parity
reference.

## Next

- [x] P1 · milestone 0: new MIT KwikCore repo with BEHAVIOUR.md, decode, peaks, transcriber library, `kwik transcribe` CLI (brief: docs/archive/NATIVE-PLAN.md §7) · added 2026-10-16
- [ ] P2 · decide whether the Electron fork also needs the ~44 ms DJI AAC-priming timing fix; shared-engine investigation belongs to KwikCore LOG.md · added 2026-10-16
- [x] P3 · fold ROADMAP.md into owner logs and archive historical plans; task dispositions below · added 2026-10-16

## Decisions

- This remains the PolyForm Noncommercial Electron fork of Rescript, with its
  attribution and license intact. The native app and KwikCore use separate MIT
  repositories under the recorded clean-room decision; this does not relicense
  the fork. Preserve the installed helper until its consumers are explicitly
  migrated and verified.
- The 2026-10-17 direction is native playback/render, with FCPXML secondary.
  KwikCore cleanup must match or improve the Electron behaviour through manual
  comparisons on the same clips. Detailed behaviour belongs to KwikCore's
  current docs, not a refreshed specification in this legacy repository.
- Native scope: free forever; text editing, cleanup, simple line/karaoke captions,
  basic audio cleanup, projects/autosave and opt-in offline proxy. Offline
  transcription remains default; Scribe is opt-in with the user's key. AI story
  editing, verticals, advanced captions and coaching belong to KwikCut/KwikLog.
- Prior Next task dispositions (2026-10-22; all were added 2026-10-16):
  - P2 · import `~/Desktop/DJI 0018 cut (kwik).fcpxml` (from `kwik cut`) in Final Cut and check relink, timecode and how tight the cuts sound → KwikCore LOG.md; open verification.
  - P2 · this app's FFmpeg WAVs keep AAC priming, so transcripts of DJI files are ~44 ms late against Final Cut (KwikCore `docs/archive/PERFORMANCE.md`); fix here or only natively? → shared timing investigation in KwikCore LOG.md, Electron decision retained above.
  - P2 · name the native app's new repo and decide whether it replaces ricardo-calcina/kwikscript → KwikScript-native LOG.md; local folder exists, public repository/replacement decision remains open.
  - P2 · decide on the Apple Developer Program for a notarized free release → KwikScript-native LOG.md; open decision.
  - P2 · pick the offline default (Parakeet v3 or Ultra) from the comparison session, and check Ultra has a Core ML build → KwikCore LOG.md; open decision.
  - P2 · test Scribe v2 keeps fillers verbatim before relying on it for cleanup (milestone 4) → KwikCore LOG.md; open verification.
  - P3 · decide whether ROADMAP.md's open items move here or it stays the public roadmap → resolved by owner logs plus intact archive.
  Milestone 1 was already recorded as built on 2026-10-17; hands-on validation remains native work. None of the open checks above is marked complete by this cleanup.
- ROADMAP dispositions (2026-10-22): personal filler vocabulary, word-timing
  accuracy and repetition/false-start detection belong to native/Core owner
  logs; richer correction, optional SpeechAnalyzer, languages/offline model
  import, faster rendering, multi-clip projects/caption burn-in and gap clips
  remain native backlog decisions. FCPXML already exists (recorded below);
  additional NLE formats require a concrete request. Voice-clone regeneration
  remains an archived idea outside current native scope; no implementation is committed. Historical
  checkboxes are preserved verbatim in docs/archive/ROADMAP.md, not presented as
  newly verified features or completed native parity.
- The old migration gates and NATIVE-PLAN draft status were superseded by the
  recorded native work. Preserve plans intact in docs/archive for reference;
  do not follow the superseded TypeScript fixture generator/source-porting
  proposal. README owns current operation; AGENTS owns implementation rules;
  DESIGN remains active. No separate maintained roadmap/spec/release runbook.

## Done

- 2026-10-27 · Old `dist` installers (dmg + zip of the 2 Sep build) archived to the SSD (`Archive/KwikScript-electron-dist`), checksums matched, Mac copy trashed · SSD ARCHIVED.tsv
- 2026-10-24 · Shared design contract records native Natural/Tight pacing and complete-thought clips; implementation and validation belong to the native app · DESIGN.md; ../KwikScript-native/LOG.md

- 2026-10-24 · Updated active GitHub references for ricardo-calcina; app identifiers and historical evidence preserved · GitHub username migration

- 2026-10-22 · grouped whole project with its siblings under Content Creation; preserved project boundaries and existing work · ../README.md

- 2026-10-22 · Trashed node_modules and copied public/vendor; retained old dist installers, authored icons and installed app/helper · README.md; package-lock.json; scripts/copy-assets.mjs
Historical `NATIVE-PLAN.md` and `docs/PERFORMANCE.md` references below resolve to their originals under `docs/archive/`; the named KwikCore measurements are now at `../KwikCore/docs/archive/PERFORMANCE.md`.

- 2026-10-22 · consolidated operations/rules/log, archived historical plans intact and assigned remaining work to owner logs; kept Electron helper dependency and license boundary · README.md, AGENTS.md, docs/archive/

- 2026-10-17 · milestone 1 native app built at `../KwikScript-native` (MIT, no commits yet): opens, transcribes, cleans up, edits by word, plays gaplessly and exports MP4/FCPXML with no Electron; idle 120 MB against this app's ~452 MB on the same 20 min file; waiting on a hands-on run · KwikScript-native LOG.md
- 2026-10-16 · KwikCore ahead of plan: edit model, filler/pause/quiet cleanup and FCPXML 1.11 export with source timecode; `kwik cut` takes clip 0018 from 9:28.9 to 8:24.2 (25 fillers, 175 pauses) · KwikCore c88e3ea
- 2026-10-16 · milestone 0 done in KwikCore (committed locally): `kwik transcribe` output identical to the bundled helper on 3 DJI clips, AVFoundation decode 2.1 s / 30 MB against FFmpeg 0.6 s / 22 MB on 9.5 min 4K · KwikCore LOG.md, docs/PERFORMANCE.md
- 2026-10-16 · confirmed the opt-in offline proxy (540p copy in the cache, originals for render and FCPXML) · NATIVE-PLAN.md §3
- 2026-10-16 · added projects (Projects window, autosave, opt-in offline proxy) to the plan; milestone 0 started · NATIVE-PLAN.md §3, §5
- 2026-10-16 · scope revised: Descript replacement, free forever; online Scribe v2 opt-in, simple burn-in captions, basic audio clean-up, video export in milestone 1; EDIT-THIS.md superseded (moved to KwikCut/KwikLog), EditCore folded into KwikCore · NATIVE-PLAN.md
- 2026-10-16 · chose licence option A (MIT, new repos, clean-room from behaviour); third-party licences checked, all permissive · NATIVE-PLAN.md §4
- 2026-10-16 · wrote the native rebuild plan: feature inventory, KwikCore API, app architecture, licence options, milestones · NATIVE-PLAN.md
