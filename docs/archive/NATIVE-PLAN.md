# KwikScript native: plan

Rebuild KwikScript as a native macOS app in SwiftUI, replace the Electron/Next.js shell, and release it free. Lift the transcription, waveform and edit model into a Swift package, KwikCore, that KwikLog can depend on too.

Written 2026-10-16. Status: **draft, nothing built**. Licence: option A, MIT in new repos (section 4). Scope revised 2026-10-16 (below).

**Scope.** KwikScript is a Descript replacement, better than Rescript: open a video, transcribe it, edit by text, remove silences and fillers, add captions, and export a finished video or FCPXML for Final Cut. It's one app in the Kwik family (KwikLog's `ROADMAP.md`) and a finish-off project: make it native, optimise it lightly, release it free forever. The lasting output is KwikCore, which KwikLog, KwikCut and KwikCast build on; KwikScript is the building block that led to KwikCut. AI story cuts, verticals, face tracking, titles/thumbnails and recording coaching belong to KwikCut and KwikLog, not here; `docs/EDIT-THIS.md` is superseded for that reason.

**Decided 2026-10-16:**
- **Transcription:** offline by default (Parakeet v3, or Moondream's Parakeet Ultra if its test wins). Online mode with ElevenLabs Scribe v2 is opt-in, on the user's own key.
- **Captions:** styled burn-in, two styles (clean line, word-highlight karaoke), 16:9 only, previewed live and written to FCPXML and SRT/VTT too. That's the ceiling here: templates, animation and in-depth caption work belong to KwikCut.
- **Audio:** basic clean-up: fades at joins, -14 LUFS loudness with a true-peak limit, high-pass, and one "Clean voice" toggle.
- **Video export** lands in milestone 1, so the first slice can publish.

**Revised 2026-10-17:** the main path is AI editing with our own player and renderer, no NLE. FCPXML stays as a secondary export (already written in KwikCore, a nice-to-have from here). KwikCore's next job is playback and render, and its cleanup must match this app's word and silence cutting before improving on it, judged by running both on the same clips.

Sources: `README.md`, `ROADMAP.md`, `PLAN.md`, `DESIGN.md`, `docs/SWIFT-MIGRATION.md`, `docs/PERFORMANCE.md`, `docs/EDIT-THIS.md`, `app/`, `components/`, `lib/`, `electron/`, `native/coreml-transcriber/`, git blame, KwikLog `DESIGN.md`, and the 2026-10-16 transcription compare (KwikScript's transcriber beat Borumi: no dropped speech, aligned word timings, ~40× realtime offline).

**Why now, when `SWIFT-MIGRATION.md` said wait:** that doc gates a rewrite on Electron performance problems. The reason has changed: I want a free release without an Electron/FFmpeg install story, and KwikLog needs the same core in Swift. The gate doc's "one vertical slice first" advice still holds, and is milestone 1 below.

---

## 1. Feature inventory

### What the app does today

| Area | What it does | Where |
|---|---|---|
| Ingest | Open a video or audio file in place (no copy). Native FFmpeg extracts 16 kHz mono WAV, twice (once for the transcriber, once for the waveform). H.264 viewing proxy when Chromium can't play the codec (ProRes) | `electron/main.ts`, `lib/source.ts`, `lib/media.ts` |
| Transcription | Swift helper: Parakeet TDT 0.6b v3 int8 on the Neural Engine, Parakeet CTC 110m acoustic word alignment, Silero VAD placeholders (`...`) for vocalisations Parakeet skipped. JSON out, progress on stderr, cancel | `native/coreml-transcriber/` |
| Legacy transcription | Browser Whisper (transformers.js, WebGPU/WASM), JS alignment, pyannote diarization, hallucination filters, model selector | `workers/`, `lib/align.ts`, `lib/forcedAlign.ts`, `lib/diarize.ts`, `lib/models.ts`, `lib/webgpu.ts`, `lib/hallucinations.ts` |
| Transcript editor | Click a word to seek (play or seek-only toggle), select words, cut/restore, correct a word (timings redistributed), mark as filler, playhead follow, scroll rail with ticks | `components/TranscriptPanel.tsx`, `hooks/useTranscript*` |
| Timeline | Waveform, ruler, word chips, cut regions, playhead, zoom/pan; drag word start/end, split at playhead, clip trim handles, draggable cut edges, scrub | `components/Timeline.tsx`, `lib/edits.ts`, `lib/waveform.ts` |
| Cleanup sidebar | Three tools, one preview layer at a time: quiet audio (waveform threshold, never cuts a word, grey), filler words with a personal word/phrase library (red), transcript pauses (amber). Duration "Up to" / "Between", linked or split left/right padding, true zero padding, remembered settings per tool, remove and restore | `components/CleanupSidebar.tsx`, `lib/silences.ts`, `lib/fillers.ts`, `lib/disfluencies.ts`, `lib/*Preferences.ts` |
| Edit state | Word list with `deleted`, manual cuts with an owner (`silence`), scene boundaries, undo/redo snapshots | `lib/types.ts`, `lib/store.ts` (zustand) |
| Playback | `<video>` that skips cut ranges in real time | `components/MediaPreview.tsx`, `lib/playback.ts` |
| Export | Video MP4/WebM, audio M4A/MP3/WAV (native FFmpeg); transcript TXT/MD; subtitles SRT/VTT/JSON; timeline FCPXML, Premiere XML, Resolve, AAF | `components/ExportDialog.tsx`, `lib/serializeTimeline.ts`, `lib/serializeTranscript.ts`, `lib/aaf/` |
| Import | Own transcript (SRT/VTT/JSON) instead of transcribing | `lib/parseTranscript.ts` |
| Projects | Autosave to IndexedDB, recents in the native menu | `lib/projects.ts`, `lib/autosave.ts`, `electron/menu.ts` |
| Shell | Nine UI locales, light/dark, collapsible panes, Sentry + opt-in telemetry, auto-updater, social links, web/mobile layouts | `electron/`, `lib/i18n/`, `lib/telemetry.ts` |

### Keep, change, drop

**Keep (the reason the app exists):**
- Native transcription, unchanged in behaviour: Parakeet v3 + CTC alignment + Silero placeholders.
- Transcript-as-editor: click to seek, select, cut/restore, correct a word, mark as filler, follow the playhead.
- Timeline: waveform, word chips, cuts, zoom/pan, word-boundary drag, split, trim handles, cut-edge drag.
- The three cleanup tools with previews, ranges, padding, remembered settings and restore. Words stay protected from quiet-audio cuts.
- Undo/redo.
- FCPXML export for Final Cut.
- Video and audio export, transcript TXT/MD, SRT/VTT. Rendered video becomes a main output, equal to FCPXML.
- Colours and layout from `DESIGN.md` (zinc chrome, red/amber/grey semantics, three columns over a full-width timeline).

**Change:**
- Media decode: AVFoundation instead of Homebrew FFmpeg, so the free build has no prerequisite. `PERFORMANCE.md` measured the AVFoundation experiment ~10% slower (15.5 s vs 14.0 s on a 9-minute 4K file) at 94.6 MB vs 22.5 MB. Both are fine on 8 GB, and decoding once (not twice) wins back most of the time. Re-measure with `AVAssetReader` asking for 16 kHz mono Float32 directly, read in chunks.
- Projects: a `.kwik` file beside the media instead of IndexedDB. Caches (PCM, peaks) in `~/Library/Caches/KwikScript/`.
- Edit state: operation-based with owners (see section 2) instead of a mutable `deleted` flag plus side lists. Same behaviour, cleaner restore, and the same model KwikLog needs.
- JSON import: KwikScript's own JSON only (it has word timings). SRT/VTT import has no word timings, so it's dropped.
- Transcription engines: one protocol, several engines. Parakeet stays the offline default; Scribe v2 is an opt-in online engine; Parakeet Ultra is added only if it beats v3 and runs under Core ML (section 3).

**Add (new for Descript parity):**
- Styled captions burned into the video, following the edited transcript (section 3).
- Basic audio clean-up on export (section 3).

**Drop:**
- Web build, GitHub Pages, Vercel, COOP/COEP, ffmpeg.wasm, transformers.js Whisper, WebGPU, ONNX, the JS aligners, hallucination filters, the model selector.
- Diarization and speaker controls (already disabled). The `Word` type keeps an optional speaker for later.
- Viewing proxies: AVPlayer plays ProRes and HEVC natively.
- Sentry and telemetry. The only network use is the one-time model download, plus Scribe uploads when online mode is switched on.
- Nine locales for v1. English only, written with String Catalogs so translations can come back.
- Premiere XML, Resolve XML and AAF. Resolve imports FCPXML. Re-add XMEML if someone asks.
- Social links, desktop-app banner, mobile layout, Windows, Linux, Intel.
- WebM and MP3 export.
- Electron auto-updater (see milestone 6 for what replaces it).

---

## 2. Core boundary: KwikCore

### What goes in

| Module | Contents | Comes from |
|---|---|---|
| `KwikMedia` | Time and range types, media probe, source fingerprint, AVFoundation PCM decode to a file-backed cache, multi-resolution waveform peaks | new; peaks reuse the envelope code in `AcousticAlignment.swift` |
| `KwikTranscribe` | The `TranscriptionEngine` protocol. Parakeet engine: model download/compile, transcription, CTC alignment, Silero placeholders, progress, cancel, unload | `native/coreml-transcriber` as it is, made into a library. FluidAudio pinned at 0.15.6 |
| `KwikTranscribeCloud` | Scribe v2 engine over HTTPS: compressed audio upload, word timings back, mapped to `Word`. Key from the Keychain, passed in by the app. No SDK. KwikLog adds its Gemini-words pass on top here | new |
| `KwikEdit` | Word/transcript model, operations with owners, derived cuts/keep ranges/clips, source↔edited time mapping, the three cleanup detectors, undo-friendly value types, document schema + migration | new Swift, designed from `BEHAVIOUR.md` with hand-written fixtures (section 4); no upstream code |
| `KwikInterchange` | FCPXML writer, SRT/VTT/TXT/MD/JSON writers | new |
| `KwikPlayback` | Build an `AVComposition` from a sequence, render/export with VideoToolbox, audio mix (fades, loudness, high-pass, clean voice) | new |
| `KwikCaptions` | Caption lines from an `EditState` + `Transcript` (line breaking, edited timing), styles, and Core Animation layers used both over the player and for burn-in | new; KwikCut reuses it |
| `kwik` CLI | `kwik transcribe`, `kwik peaks`, `kwik export` | replaces `rescript-coreml-transcriber` |

Products: `KwikCore` (Media + Edit + Interchange, no ML, no FluidAudio, no network), `KwikTranscribe`, `KwikTranscribeCloud`, `KwikPlayback`, `KwikCaptions`, and the `kwik` executable. KwikLog takes what it needs, for example `KwikCore` + `KwikTranscribe` + `KwikTranscribeCloud` and its own renderer. Network code lives only in `KwikTranscribeCloud`, so the offline products can be checked for it.

**Rules for the package:**
- No UI and no SwiftUI. AppKit only in `KwikPlayback` if needed.
- Everything public is `Sendable` and value-typed except the transcription engines (actors).
- Time is `MediaTime`: Int64 ticks at 240,000 per second. Every common frame rate (23.976, 24, 25, 29.97, 30, 50, 59.94, 60) and the 16 kHz and 48 kHz sample rates land on whole ticks, so cuts don't drift. `CMTime` only at the AVFoundation boundary, and frame-rational time only in the FCPXML writer. Never serialise `Infinity` (from `EDIT-THIS.md`).
- The document is versioned JSON with a migration step from day one.

### The API it exposes

```swift
// KwikMedia
public struct MediaTime: Hashable, Comparable, Codable, Sendable   // ticks @ 240_000/s
public struct TimeRange: Hashable, Codable, Sendable               // half-open [start, end)
public struct SourceID: Hashable, Codable, Sendable
public struct SourceMedia: Codable, Sendable {                     // url, bookmark, fingerprint,
    public static func probe(_ url: URL) async throws -> SourceMedia   // duration, fps, size, hasVideo
}
public struct PCMFile: Sendable {                                  // 16 kHz mono Float32, mmap'd
    public static func decode(_ source: SourceMedia, cacheDirectory: URL,
                              progress: @Sendable (Double) -> Void) async throws -> PCMFile
}
public struct WaveformPeaks: Codable, Sendable {
    public static func build(from pcm: PCMFile) throws -> WaveformPeaks   // 10 ms base, ×4 mip levels
    public func minMax(in range: TimeRange, buckets: Int) -> [ClosedRange<Float>]
    public func rms(in range: TimeRange) -> Float
}

// KwikTranscribe
public protocol TranscriptionEngine: Actor {
    var info: EngineInfo { get }                                       // id, name, offline?, model
    func prepare(progress: @Sendable (ModelProgress) -> Void) async throws
    func transcribe(_ pcm: PCMFile,
                    progress: @Sendable (TranscriptionProgress) -> Void) async throws -> Transcript
    func unload()
}
public actor ParakeetEngine: TranscriptionEngine {                    // was Transcriber
    public init(_ configuration: ParakeetConfiguration = .init())     // model (v3, Ultra), language, compute units
}

// KwikTranscribeCloud
public actor ScribeEngine: TranscriptionEngine {
    public init(apiKey: String, configuration: ScribeConfiguration = .init())   // model, language
}

// KwikEdit
public struct Word: Codable, Identifiable, Sendable {
    public let id: WordID; public var text: String; public var range: TimeRange
    public var kind: Kind            // .spoken, .vocalisation (Silero placeholder)
}
public struct Transcript: Codable, Sendable { revision, source, words, language, model }
public struct EditOperation: Codable, Identifiable, Sendable {
    public var owner: Owner          // .human, .fillers, .pauses, .quiet, .agent(String)
    public var action: Action        // .cutWords([WordID]), .cutRange(TimeRange),
                                     // .moveWordBounds(WordID, TimeRange), .split(MediaTime),
                                     // .correctWord(WordID, String)
    public var reason: String?; public var parameters: [String: Double]; public var enabled: Bool
}
public struct EditDocument: Codable, Sendable {   // the .kwik file
    public var schemaVersion: Int
    public var sources: [SourceMedia]; public var transcripts: [Transcript]
    public var operations: [EditOperation]; public var sequences: [Sequence]
}
public struct Sequence: Codable, Identifiable, Sendable { clips: [ClipRef] }   // ordered
public struct ClipRef: Codable, Sendable { source: SourceID; range: TimeRange }
public struct EditState: Sendable {               // derived, never stored
    public init(document: EditDocument, source: SourceID)
    public var cuts: [TimeRange]; public var keepRanges: [TimeRange]; public var clips: [ClipRef]
    public var editedDuration: MediaTime
    public func editedTime(for source: MediaTime) -> MediaTime
    public func sourceTime(for edited: MediaTime) -> MediaTime
    public func isCut(_ word: WordID) -> Bool
}
public enum Cleanup {
    public static func fillers(_ t: Transcript, vocabulary: FillerVocabulary) -> [Candidate]
    public static func pauses(_ t: Transcript, _ s: PauseSettings) -> [Candidate]
    public static func quietAudio(_ p: WaveformPeaks, protecting t: Transcript, _ s: QuietSettings) -> [Candidate]
}

// KwikInterchange
public enum FCPXML { public static func write(_ seq: Sequence, sources: [SourceMedia], frameRate: FrameRate) throws -> Data }
public enum Captions { public static func srt(_ state: EditState, _ t: Transcript) -> String; vtt; text; markdown }

// KwikPlayback
public enum Composition { public static func build(_ seq: Sequence, sources: [SourceMedia]) async throws -> AVComposition }
public enum Render { public static func export(_ seq: Sequence, sources: [SourceMedia], to: URL,
                                              preset: RenderPreset, audio: AudioCleanup,
                                              captions: CaptionTrack?, progress: ...) async throws }
public struct AudioCleanup: Codable, Sendable { fades, targetLUFS (-14), truePeak (-1 dBTP), highPass, cleanVoice }

// KwikCaptions
public struct CaptionStyle: Codable, Sendable { kind: .line | .karaoke; font, size, colours, position, maxLines, maxChars }
public struct CaptionTrack: Codable, Sendable {       // derived from the edit, line breaks overridable
    public init(_ state: EditState, _ t: Transcript, style: CaptionStyle, overrides: [LineBreakOverride])
    public var lines: [CaptionLine]                   // edited-time ranges, word ranges for karaoke
}
public enum CaptionLayers { public static func make(_ track: CaptionTrack, size: CGSize) -> CALayer }
```

Effective cuts come from enabled operations, so turning one off never disturbs another that overlaps it. KwikScript uses one source and one sequence derived from keep ranges. KwikLog uses many sources and builds its own ordered sequences for the story cut. B-roll, icons and sound stay in KwikLog as layers on top of a `Sequence`; KwikCore stays single-track.

### How KwikLog uses it

- Speech stage (KwikLog milestone 3): `ScribeEngine` (with KwikLog's Gemini-words pass) by default and `ParakeetEngine` as the local fallback, replacing the `tmux-whisper` subprocess. One heavy job at a time still applies: KwikLog's serial heavy lane calls `transcribe` and then `unload`.
- Story rough cut: sentence ranges from `Transcript`, then `Sequence`/`ClipRef` for the cut list, and `Cleanup.fillers`/`pauses` to tighten each clip.
- Preview: `Composition.build` (KwikLog already plans an AVComposition preview).
- Depend by local path while both are in development (`.package(path: "../KwikCore")`), by git tag once stable.
- KwikLog's `DESIGN.md` says "nothing from KwikScript (PolyForm Noncommercial)". With KwikCore under MIT that rule still holds: KwikLog depends on KwikCore, never on this repo. Update that line when KwikLog adds the dependency.

### Where it lives

A separate repo, `~/Documents/Personal Projects/KwikCore`, so its licence and history are separate from the Rescript fork and KwikLog can depend on it without pulling KwikScript. Moving `native/coreml-transcriber` there with `git filter-repo --path native/coreml-transcriber` keeps its 5 commits, all mine. The native app gets its own new repo too (name to decide). Both MIT.

### Relation to `docs/EDIT-THIS.md`

That plan is superseded (2026-10-16): its AI editing, verticals, coaching and analytics moved to KwikCut and KwikLog. Its `EditCore` (manifest, ticks, owned operations, cleanup logic) is `KwikEdit` here, so that part is folded in. The `.agent(String)` owner and the `kwik` CLI stay, so KwikCut or Claude can drive the same edit model later, but KwikScript builds no AI editing of its own.

---

## 3. App architecture

```
KwikScript.app (SwiftUI, macOS 14+, Apple Silicon)
 ├─ ProjectWindow (one per .kwik document)
 │   ├─ CleanupSidebar   SwiftUI
 │   ├─ TranscriptView   NSTextView (TextKit 2) in NSViewRepresentable
 │   ├─ PlayerView       AVPlayerView over an AVComposition
 │   └─ TimelineView     Canvas (Metal-backed layer if frame time demands)
 ├─ ProjectModel (@Observable, main actor): EditDocument + UndoManager + derived EditState
 ├─ JobQueue: one heavy job at a time (decode, transcribe, render)
 └─ Contents/Helpers/kwik   (the KwikCore CLI, runs transcription out of process)
```

**Transcription out of process.** The app runs the bundled `kwik transcribe` as a child process, as Electron does now. Core ML model memory (~0.5-1 GB) is gone the moment it exits, and a Core ML crash can't take the editor down. Later this can become an XPC service. Library users like KwikLog can call the engines in process if they prefer.

**Playback.** Build an `AVMutableComposition` from the current keep ranges and play it with `AVPlayer`. Playback across cuts is gapless, unlike seeking past cut ranges. On each edit, rebuild the composition off the main thread, swap the player item and keep the playhead using `EditState.editedTime(for:)`. Previews (red/amber/grey candidates) are overlays only; they stay playable until committed, as now. Risk: a rebuild hitch with hundreds of cuts. Measure in milestone 1, and fall back to rebuilding only the changed span if it shows.

**Export.** Video goes through `AVAssetWriter` with VideoToolbox (H.264 or HEVC, 1080p or source size), and audio to M4A/WAV. Audio always comes from the source file at full quality, never from the 16 kHz transcription PCM. FCPXML references the original media path, so Final Cut relinks to the 4K original, and carries the captions as a caption track (text and timing, not the burned-in style).

**Transcription engines.** A per-project choice in the transcribe sheet, remembered as the default:
- **Offline (default): Parakeet.** v3 as today. Moondream's Parakeet Ultra (same 0.6B architecture, post-trained) is added as a second model only if the running comparison shows it's better and a Core ML build exists or can be converted. Runs out of process as above.
- **Online (opt-in): ElevenLabs Scribe v2.** Off until a key is pasted in Settings; the key lives in the Keychain. The app uploads a compressed mono audio track (not the video) and maps the returned words and timings to `Word`. Scribe's own timings are used, with no CTC alignment pass. The sheet says that audio leaves the Mac and shows a rough cost from the duration. Runs in process (it's just HTTPS).
- **Must test before relying on Scribe:** that it keeps fillers ("um", "uh", false starts) verbatim, because filler removal depends on it; how it marks non-speech (audio-event tags map to `.vocalisation`); and its timing at cut points against Parakeet + CTC on `test2.mp4`.
- The transcript records the engine and model, so a re-transcribe with the other engine is a fresh transcript revision, never a silent swap.

**Captions.** Derived from the edit, so cutting words updates them with no extra step:
- `CaptionTrack` builds lines from the kept words in edited time. Line breaking is automatic (max characters and lines from the style), with manual break overrides stored in the document by word id. Text corrections come from `.correctWord`, so the transcript and captions never disagree.
- Two styles in v1: **line** (plain subtitle, one or two lines) and **karaoke** (the current word highlighted). Font, size, text and highlight colours, background box, and position (bottom/middle/top plus offset). 16:9 only; vertical reframing is KwikCut's job.
- Preview: the caption layers sit in an `AVSynchronizedLayer` over the player, so they track playback with no re-render. Burn-in: the same layers through `AVVideoCompositionCoreAnimationTool` on export. One layer builder for both, so preview matches output.
- A sidebar tab (beside Cleanup) for style and a captions on/off switch; per-line edits happen in the transcript.
- Also exported as SRT/VTT and as the FCPXML caption track.

**Audio clean-up.** Applied on export and in a preview toggle, settings saved per project:
- Fades of ~10 ms at every cut join, always on (clicks otherwise).
- High-pass at ~80 Hz, on by default.
- Loudness: measure the edited programme (ITU-R BS.1770 integrated loudness, a small own implementation), then gain to -14 LUFS with a -1 dBTP limiter. This needs a quick analysis pass before the render; the result is cached by edit revision.
- **Clean voice** toggle: try Apple's sound isolation audio unit first (check it runs offline on file audio and how it sounds on my recordings). Only if it isn't good enough, bring in DeepFilterNet (permissive licence, check it then) as a downloaded model. No de-reverb or EQ chains in v1.

**Waveform drawing.** Peaks are built once from the PCM cache (10 ms min/max with ×4 levels, ~2-4 MB for an hour) and stored as a binary file next to it. The timeline draws only the visible range from the closest level with SwiftUI `Canvas`: waveform, word chips, cut hatching, preview colours, playhead. Playhead and scrub redraw in a separate layer so the waveform isn't redrawn at 60 fps. Target: 60 fps zoom/pan on a 60-minute file on the M1 Air; move the waveform layer to Metal only if `Canvas` misses that.

**Transcript as editor.** One `NSTextView`, read-only for typing, with a custom attribute holding each word's id:
- Selection maps to a word-id range. ⌫ adds a `.human` cut operation instead of deleting text.
- Cut words get strike-through, and previews get background colours, as in `DESIGN.md`.
- The playhead word is highlighted with TextKit 2 rendering attributes, so moving it doesn't re-layout the text.
- Click to seek, with the play / seek-only toggle. A header bar holds Cut, Correct and Mark as filler. Correct opens a small field; `.correctWord` keeps the timing.
- ~10k words for an hour fits TextKit comfortably.

**Media stays in place.**
- The `.kwik` file sits beside the source and stores the path, a bookmark and a fingerprint (size, mtime, hash of the first and last MB).
- When the file moved, the bookmark finds it. When the fingerprint differs, the app asks before reusing the transcript.
- Missing media shows a relink prompt and never deletes anything.
- PCM, peaks and transcripts are caches under `~/Library/Caches/KwikScript/<fingerprint>/`, safe to delete.
- No copies. Proxies only as the opt-in below.
- Not sandboxed (direct download, not the App Store), so the app reads files by path.

**Projects** (added 2026-10-16; today's Electron app is one working file at a time, with 10 autosaves in IndexedDB):
- A project is the `.kwik` document: transcript, operations, caption style, cleanup and audio settings. Autosaved on every edit, so closing and reopening picks up exactly where I left off. Several can be open at once, one window each.
- A **Projects window** on launch: recent projects with a poster frame, duration, edited duration, last opened, and a "media offline" badge when the source drive isn't mounted. Search by name. Removing a project from the list never deletes the `.kwik` or the media.
- **Offline proxy (opt-in per project, confirmed 2026-10-16):** a small 540p HEVC copy in `~/Library/Caches/KwikScript/<fingerprint>/proxy.mov`, so a project whose footage is on the SSD still opens, plays and can be edited with the drive unplugged. Renders and FCPXML always use the original and ask for the drive when it's missing. The transcript, waveform and edits never need the proxy, so the cost is disk space only (roughly 100-150 MB an hour).

**Build without Xcode.** This Mac has only the Command Line Tools (Swift 6.4). The app is a SwiftPM executable plus a script that assembles `KwikScript.app`, as KwikLog plans. No SwiftUI previews and no Instruments GUI, so measure with `/usr/bin/time -l`, `os_signpost` and a debug frame-time overlay. Builds are heavy jobs on this machine: never build while transcribing or rendering.

---

## 4. Licence: option A (decided 2026-10-16)

Not legal advice. These are the facts from the repo and how option A works.

**Decision:** KwikCore and the native app are new code I own, in new repos, under MIT. They're built from what the app should do and produce, not from how Rescript's code does it. That gives free choice later on redistribution and gating, even though I don't plan a paywall. The Electron KwikScript in this repo stays PolyForm Noncommercial with the Rescript notice, unchanged, until it's archived.

Option A was chosen over:
- **B:** stay PolyForm. It can't be called open source, and it limits KwikLog.
- **C:** ask Wassim to relicense the edit logic.

**The facts behind it:**
- This repo is PolyForm Noncommercial 1.0.0, copyright Wassim Gharbi and Rescript contributors, with a `Required Notice:` line. It is not an OSI open-source licence.
- Git: all 24 commits since the fork point are mine. The 122 "Cursor Agent" commits are upstream's.

**Mine, so it moves to the new repos:**

| What | Evidence |
|---|---|
| `native/coreml-transcriber/` (`AcousticAlignment.swift` 645 lines, `main.swift` 383, tests 217, `Package.swift`) | git blame 100% mine, 5 commits from 2026-09-02 to 09-03. A separate program with no Rescript code. Its output is Rescript's six-field `Word` JSON shape, which KwikCore replaces with its own type, keeping a compatibility writer only for the Electron app |
| `DESIGN.md` | 115 of 115 committed lines mine. The colour and layout semantics carry over |
| The D02 app icon and the name KwikScript | mine |

**Not carried over (upstream code; behaviour only):** `lib/edits.ts`, `store.ts`, `silences.ts`, `fillers.ts`, `waveform.ts`, `Timeline.tsx`, `TranscriptPanel.tsx`, `serializeTimeline.ts`, `serializeTranscript.ts`, `aaf/`, the JS aligners, `electron/`. My TypeScript inside upstream's app (`CleanupSidebar.tsx`, the preference files, `parakeetRecovery.ts`) isn't ported either. The Swift versions are designed fresh.

**How the new code stays clean:**
1. **The spec is behaviour.** KwikCore gets a `BEHAVIOUR.md` that says what each tool does and outputs. Examples: "quiet audio never cuts through a recognised word", "padding can be zero", "restoring one tool's cuts leaves the others alone". It's written from my README, `DESIGN.md` and how I use the app, not from reading upstream source.
2. **Agents building KwikCore or the native app don't open** upstream files in this repo (`lib/`, `components/`, `hooks/`, `electron/`, `workers/`) as reference. My own `native/` Swift is fair game.
3. **Fixtures are written by hand** from that spec. Comparing against the old app happens as a user: open the same file in both, compare cut counts and ranges, and explain each difference. Where the native result is better, keep it. The `EDIT-THIS.md` idea of generating fixtures by running the TypeScript is dropped.
4. **Architecture is free to differ:** owned operations instead of a `deleted` flag, integer ticks instead of float seconds, composition playback instead of skip-seeking.
5. **Credit as courtesy, not obligation:** a README line like "KwikScript started as a fork of Rescript by Wassim Gharbi".

**Third-party parts:** all permissive, all allow MIT and commercial use. Checked 2026-10-16.

| Part | Licence | What I have to do |
|---|---|---|
| FluidAudio 0.15.6 (Swift package, compiled in) | Apache-2.0 | Ship its `LICENSE` and its `ThirdPartyLicenses/` folder (NeMo text processing, fastcluster, VBx) in the app's Acknowledgements. No NOTICE file to carry. Say so if I ever patch it |
| Parakeet TDT 0.6b v3 Core ML (`FluidInference/parakeet-tdt-0.6b-v3-coreml`, from `nvidia/parakeet-tdt-0.6b-v3`) | CC-BY-4.0 (both cards) | Credit NVIDIA and FluidInference with a licence link in About and the README |
| Parakeet CTC 110m Core ML (`FluidInference/parakeet-ctc-110m-coreml`, from `nvidia/parakeet-tdt_ctc-110m`) | CC-BY-4.0 (both cards) | same credit |
| Silero VAD Core ML (`FluidInference/silero-vad-coreml`) | MIT | include the notice |
| ElevenLabs Scribe v2 (online mode) | API, not a licence: nothing is bundled, plain HTTPS, each user's own key and account | Say in the app and README that online mode sends audio to ElevenLabs under their terms |
| Parakeet Ultra (Moondream, only if adopted) | CC-BY-4.0 (per the comparison session; check the card) | Credit as for v3 |
| DeepFilterNet (only if Apple's sound isolation falls short) | check when adopted | - |

The models aren't bundled. FluidAudio downloads them from Hugging Face at first run, so the app never redistributes them. That's the "download the model yourself" pattern other apps use, and it keeps the DMG small. If I want, a first-run screen can name each model, its size and its licence before the ~560 MB download starts. That's polite rather than required.

Nothing else needs doing. If a future model has a non-commercial or custom licence (FluidAudio's Sortformer diarizer is under NVIDIA's Open Model License, for example), check it before adding it.

---

## 5. Milestones

Smallest useful first. "Days" are focused working days with an agent on this 8 GB M1 Air, including build time (a release build with FluidAudio is slow here) and testing on real footage. One heavy job at a time throughout.

| # | Milestone | Useful because | Done when | Rough effort |
|---|---|---|---|---|
| **0** | **KwikCore extracted.** New MIT repo with `BEHAVIOUR.md`, `KwikMedia` (AVFoundation decode, peaks), `KwikTranscribe` (`TranscriptionEngine` protocol, current transcriber as `ParakeetEngine`), `kwik transcribe` CLI writing today's JSON shape | The Electron app keeps working on it, and KwikLog can use the transcriber straight away | Electron KwikScript runs on `kwik transcribe`; alignment tests pass; the transcript of `test2.mp4` matches the `PERFORMANCE.md` checks (309 words, five `um`, final word at 103.12 s); AVFoundation decode time and memory re-measured on the 9-minute 4K file; KwikLog builds with it as a path dependency | 2-3 days |
| **1** | **Vertical slice** (the one `SWIFT-MIGRATION.md` describes). Open a file, transcribe, waveform + word timeline, transcript view, cut/restore words, undo, play across cuts, a plain H.264 render (source audio, fades at joins), FCPXML as a secondary export, `.kwik` file beside the media with autosave and Open Recent | I can do a real talking-head cut and either hand off to Final Cut or publish the render, without Electron | A real 15-20 minute recording cut and rendered to MP4 with no clicks or drift at joins, its cut list matching or beating the Electron app's on the same file; idle memory and composition-rebuild time measured against Electron on the same file | 6-8 days |
| 2 | **Cleanup tools.** `KwikEdit` detectors with fixtures, sidebar with the three tools, one preview layer, ranges, padding, remembered settings, personal filler vocabulary, restore by owner | It's the reason I use KwikScript over Final Cut alone | `BEHAVIOUR.md` fixtures pass. On `test2.mp4` and one long recording, results are at least as good as the Electron app, with each difference explained; quiet audio never cuts a word | 4-6 days |
| 3 | **Timeline editing.** Word-boundary drag, split at playhead, trim handles, cut-edge drag, scrub, zoom/pan | Fixes the alignment misses by hand, as now | 60 fps zoom/pan on a 60-minute 4K project; memory stays bounded on it (also checks the full-PCM alignment risk from `EDIT-THIS.md`) | 4-6 days |
| 4 | **Transcription choices.** `KwikTranscribeCloud` with `ScribeEngine`, key in Settings/Keychain, engine picker in the transcribe sheet, re-transcribe with the other engine; Parakeet Ultra as a second offline model if the comparison chose it | Better accuracy when I'm online, offline stays the default | On `test2.mp4` and one long recording: Scribe keeps the fillers (or the gap is documented and filler detection adjusted), cut-point timing is within Parakeet + CTC's | 2-3 days |
| 5 | **Captions and audio.** `KwikCaptions` (line and karaoke styles, live preview, burn-in, line-break overrides, captions sidebar), audio clean-up (high-pass, -14 LUFS / -1 dBTP, Clean voice toggle), HEVC, SRT/VTT/TXT/MD, M4A/WAV, Correct word, Mark as filler, KwikScript JSON import, light/dark, the Projects window and the opt-in offline proxy | Descript parity: a finished, captioned, clean-sounding video without Final Cut; the Electron app can be retired | A 20-minute captioned render finishes without thermal throttling hurting it badly; burned-in captions match the preview frame for frame; SRT and FCPXML captions line up in YouTube and Final Cut; loudness measures -14 ±1 LUFS in a second tool | 6-9 days |
| 6 | **Free release.** Model-download first run, online-mode notice, error states, README, licence files, credits, signed and notarized DMG on GitHub Releases, update check (Sparkle, or a "new version" link) | Other people can use it | Clean install on a second user account; Gatekeeper passes; the app works fully with no key; the Electron app archived | 2-3 days, plus the decisions below |

**Total:** about 26-38 days. Milestone 0 alone is useful to KwikLog; milestones 0-1 are useful to me (cut and publish or hand off), and 0-5 replace Descript.

**Decisions needed before milestone 6** (the licence is settled):
- Notarization needs the Apple Developer Program (US$99 a year; check the UK price). Without it, the free release is "build from source" or an unsigned DMG that people have to right-click to open.
- The repo name and whether the native app replaces `ricardo-nth/kwikscript` or sits beside it.

**Risks:**
1. Composition rebuild hitches with hundreds of cuts (measured in milestone 1).
2. Long-recording memory: alignment loads the full PCM and keeps CTC probabilities. Fine at 9 minutes (166 MB), unmeasured at 60 minutes. May need chunked alignment in `KwikTranscribe`.
3. TextKit 2 quirks for a read-only, word-addressed editor. Fallback: TextKit 1 `NSLayoutManager` temporary attributes.
4. No Xcode: no Instruments or SwiftUI previews. If profiling gets stuck, installing Xcode (~10+ GB) is the fallback. The internal disk is tight, so that's a real cost.
5. AVFoundation decode is slower than FFmpeg (~1.5 s on 9 minutes). Acceptable for dropping the prerequisite, but measured in milestone 0, not assumed.
6. Brand words and number formatting (from the compare): a custom-vocabulary or post-correction pass belongs in `KwikTranscribe` later, not in v1. Scribe's keyterm support may cover this for online mode.
7. Scribe may tidy away fillers or false starts. If it does, online mode is worse for cleanup even when its words are better; test it first in milestone 4.
8. Caption burn-in through `AVVideoCompositionCoreAnimationTool` is slower than a plain export and has known quirks (layer timing must use `AVCoreAnimationBeginTimeAtZero`, no implicit animations). Measure render time on the M1 Air in milestone 5; fall back to a custom `AVVideoCompositing` that draws the same layers if it's too slow.
9. Apple's sound isolation may not run offline on file audio, or may sound processed. Then Clean voice needs DeepFilterNet and a model download, adding about 2 days.

---

## 6. Open questions for me

1. ~~Licence~~: option A, decided 2026-10-16.
2. Pay for the Apple Developer Program for a notarized free release, or ship build-from-source only?
3. ~~Fold `EDIT-THIS.md`'s `EditCore` into KwikCore~~: yes, and the rest of that plan moved to KwikCut (2026-10-16).
4. Minimum macOS: 14 as now, or 15 so the newer SwiftUI APIs come for free? (Neither affects transcription.)
5. ~~Captions, audio, online transcription, when video export lands~~: decided 2026-10-16 (top of this file).
6. Parakeet Ultra or v3 as the offline default: waits for the comparison session's result and a Core ML build.
7. `ROADMAP.md` is an existing to-do file. Fold its open items into `LOG.md`'s Next list, or leave it as the public roadmap?

---

## 7. Milestone 0 kickoff brief

Started 2026-10-16. Priority after it: milestone 1, the step from an Electron front end on a Swift back end to everything native; features after that.

**Task:** create the MIT Swift package repo KwikCore at `~/Documents/Personal Projects/KwikCore`.

1. `git init`. Bring in `native/coreml-transcriber` with its history via `git filter-repo --path native/coreml-transcriber` on a scratch clone (ask before installing git-filter-repo), or a plain copy if that fails, and say which.
2. Package layout per section 2. Products:
   - `KwikCore`: KwikMedia, plus stubs for KwikEdit and KwikInterchange. No FluidAudio.
   - `KwikTranscribe`: FluidAudio pinned exact 0.15.6.
   - The `kwik` executable.
3. Fill `KwikMedia`:
   - `MediaTime` (240,000 ticks per second) and `TimeRange`.
   - `SourceMedia.probe`.
   - `PCMFile.decode`: `AVAssetReader` straight to 16 kHz mono Float32, read in chunks.
   - `WaveformPeaks.build` and `minMax`.
4. Fill `KwikTranscribe`: the `TranscriptionEngine` protocol and actor `ParakeetEngine` (`prepare`, `transcribe`, `unload`), made from the existing transcriber. Transcription and alignment behaviour must not change. Add an empty `KwikTranscribeCloud` target so the product layout is fixed; `ScribeEngine` comes in milestone 4.
5. `kwik transcribe <audio-or-video> [out.json]` takes the same arguments and writes exactly the same JSON and stderr progress lines as today's `rescript-coreml-transcriber`, so the Electron app can use it by swapping the binary. Keep the alignment tests running.
6. Write `BEHAVIOUR.md`: what each cleanup tool, the edit model and caption line building must do and output, from the README, `DESIGN.md` and this plan. Spec only; the implementations come in milestones 2 and 5.
7. Add an MIT `LICENSE` and a README:
   - Credits: FluidAudio (Apache-2.0, plus its `ThirdPartyLicenses`), Parakeet TDT 0.6b v3 and CTC 110m (CC-BY-4.0, NVIDIA, converted by FluidInference), Silero VAD (MIT).
   - A "started as a fork of Rescript" courtesy line.

**Clean-room rule (option A):** don't open or copy from this repo's upstream code (`lib/`, `components/`, `hooks/`, `electron/`, `workers/`, `app/`). My `native/coreml-transcriber` Swift and the docs are fine.

**Machine:** 8 GB M1 Air, Command Line Tools only, Swift 6.4.
- One heavy job at a time, with release builds in the background. Other sessions build Swift and run models on this Mac too (KwikLog milestone 1, the Parakeet Ultra test): check `ps` for `swift-build`, `swift-frontend` and transcriber processes before a build or a transcription, and wait rather than overlap.
- Build with `CPLUS_INCLUDE_PATH="$(xcrun --sdk macosx --show-sdk-path)/usr/include/c++/v1"`.

**Done when:**
- `swift build` and the alignment tests pass.
- `kwik transcribe` gives the same JSON as `/Applications/KwikScript.app/Contents/Resources/native/rescript-coreml-transcriber` on a short test file: same word count, texts and timings.
- Decode time and peak memory are measured with `/usr/bin/time -l` against FFmpeg on one real file.
- A scratch package builds with KwikCore as a path dependency. Don't wire it into KwikLog yet.
- KwikCore has its own `LOG.md`.
- This repo's `LOG.md` gets one Done line, with the milestone 0 item ticked.

Commit in KwikCore only when I say so, and push nowhere.
