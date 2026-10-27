# KwikScript

KwikScript is an experimental Apple-Silicon fork of
[Rescript](https://github.com/wassgha/rescript) for fast, local talking-head
editing. It combines transcript-based cuts with the silence controls normally
found in waveform-first tools, then exports an FCPXML timeline for finishing in
Final Cut Pro.

This repository is the working Electron fork. Native development continues in
[KwikScript-native](../N3Studio/README.md), using
[KwikCore](../N3Core/README.md). See the [Kwik project map](../N3Library/README.md#n3-projects).
Keep the installed Electron app and its bundled transcriber available: KwikLog's
gym workflow still depends on that helper. A native prototype is not a reason
to remove or replace the installed app.

## What this fork adds

- Native Parakeet v3 transcription through Core ML and the Apple Neural Engine.
- Core ML acoustic word alignment so transcript selections match the visible
  waveform closely enough for safe word cuts and pause protection.
- Native Silero VAD recovery for vocalisations Parakeet heard but did not emit.
- Filler-word removal that can be previewed, removed, and restored.
- Separate transcript-pause and waveform-quiet cleanup tools. The waveform tool
  never proposes a cut through a recognized word.
- One active silence-preview layer at a time: cool gray for the optional
  waveform rough cut, amber for transcript pauses, and red for committed cuts.
- Independent remembered settings for each tool, including a manual loudness
  threshold for waveform cleanup.
- Duration modes: **Up to** a duration or **Between** two durations.
- Independent left/right padding, including true zero-padding cuts.
- Orange previews for proposed silence cuts before applying them.
- Restorable silence cuts and direct top-level cleanup actions.
- Lower-memory media ingestion that reads the original source rather than
  copying multi-gigabyte video files into the app.
- Automatic temporary H.264 viewing proxies when Electron cannot display a
  source codec such as ProRes. The original file remains the export source.
- Apple-Silicon-only packaging without the former ONNX speech runtimes.

The original text editor, waveform, media preview, manual cuts, transcript
imports, exports, and FCPXML workflow remain inherited from Rescript.

## Current scope and caveats

- macOS 14 or newer on Apple Silicon (M1 or newer).
- English and one speaker are the tested path.
- [Homebrew FFmpeg](https://formulae.brew.sh/formula/ffmpeg) is currently a
  runtime prerequisite. It is deliberately not bundled yet.
- The first transcription downloads approximately 460 MB of Parakeet v3 Core
  ML models, a 99 MB Core ML word-alignment model, and a roughly 1 MB Silero
  model. FluidAudio caches all three under the user Library, so rebuilding or
  replacing the app does not download them again.
- Local builds are ad-hoc signed. A public click-to-install binary still needs a
  Developer ID certificate and Apple notarization.
- Intel Macs, Windows, Linux, and the web build are not maintained by this fork.
- Speaker diarization is intentionally disabled for the current workflow.
- The upstream PolyForm Noncommercial license remains in force. See
  [License](#license) before using or distributing the software.

## Build it locally

Install the Apple command-line tools, Node.js 22, and FFmpeg:

```bash
xcode-select --install
brew install node@22 ffmpeg
```

Clone and build:

```bash
git clone https://github.com/ricardo-calcina/kwikscript.git
cd kwikscript
npm ci
npm run dist
```

The Apple-Silicon app and archives are written to `dist/`:

```text
dist/mac-arm64/KwikScript.app
dist/KwikScript-mac-arm64.dmg
dist/KwikScript-mac-arm64.zip
```

Open the locally built app:

```bash
open "dist/mac-arm64/KwikScript.app"
```

Because a local build is not notarized, macOS may ask you to confirm it the
first time. Do not bypass Gatekeeper for binaries from people you do not trust;
building from source is the supported route for now.

## Development

```bash
npm ci
npm run electron:dev
npm run lint
npm run typecheck:electron
npx next typegen && npx tsc --noEmit
npm run test:i18n
npm run test:timeline
npm run test:native:alignment
```

The native helper is in [`native/coreml-transcriber`](native/coreml-transcriber).
Its Swift package pins FluidAudio so model behaviour does not drift silently.

## Performance and regression checks

Build and benchmark only when the task requires it; this 8 GB M1 is shared with
other Swift builds and transcription jobs. Run one heavy job at a time.

```bash
npm run build:native:coreml
npm run benchmark:coreml -- /absolute/path/to/video.mp4
```

The benchmark uses native FFmpeg to extract mono 16 kHz WAV, then invokes the
Swift helper. `--keep-temp` retains its extracted audio and JSON; otherwise the
script removes its own temporary directory. `RESCRIPT_FFMPEG_PATH` can select
the FFmpeg executable. FluidAudio is pinned to 0.15.6 in the helper package.

Historical M1/8 GB measurements are preserved in
[the performance baseline](docs/archive/PERFORMANCE.md). The September nine-minute
numbers predate acoustic alignment; they are not a current end-to-end baseline.
The short `test2.mp4` reference is 309 timed words, five literal `um` tokens, one
Silero hesitation placeholder, final word at 103.12 s, and no long run of dots.
Check audible word boundaries as well as counts. Native timing work and current
measurements belong in [KwikCore's log](../N3Core/LOG.md).

## Packaging and release boundaries

`npm run dist` builds the Core ML helper, the desktop static export and Electron
bundle, then packages Apple-Silicon DMG/ZIP with `--publish never`. Its after-pack
hook ad-hoc signs the app. Local output does not establish notarization.

The current [release workflow](.github/workflows/release.yml) runs manually or
on version tags, verifies source and the packaged helper, and uploads unsigned
DMG/ZIP test artifacts to Actions. It does not publish a GitHub Release. Public
click-to-install distribution still requires Developer ID signing and Apple
notarization work. Windows/Linux configuration remains inherited and is not the
maintained release path.

Do not use `npm run release` or `npm run cut:*` as a local verification step:
`release` requests publication, and `cut:*` changes versions, commits, tags and
pushes. `npm run notes:preview` invokes the AI release-notes generator and needs
`AI_GATEWAY_API_KEY`; it is not an offline check. The upstream release guide is
[archived](docs/archive/RELEASING.md), including its credential inventory; its
multi-platform auto-publish description does not describe this fork's workflow.

## Editing workflow

1. Import the original video or a time-identical proxy.
2. Let Core ML generate the word-timed transcript.
3. Optionally start with **Remove quiet audio** for a waveform-led rough cut of
   obvious dead space. Tune threshold, duration, and padding while the cool-gray
   candidates remain audible.
4. Preview and remove filler words; this exits the waveform candidate view and
   moves into transcript cleanup.
5. Preview transcript pauses, then remove only the duration range you do not
   want.
6. Restore any intentional pauses or quiet sections as needed.
7. Export FCPXML and reconnect to the original 4K media in Final Cut Pro.
8. Finish colour, captions, graphics, and delivery in Final Cut.

## Documentation

[LOG.md](LOG.md) owns this repository's current work, decisions and history.
[AGENTS.md](AGENTS.md) owns agent rules; [DESIGN.md](DESIGN.md) remains the active
UI context. Future native app work belongs in [the native log](../N3Studio/LOG.md),
and shared engine work in [the core log](../N3Core/LOG.md).

[Archived plans](docs/archive/) preserve historical decisions, upstream features,
measurements and migration proposals. They are reference material, not a second
maintained roadmap. In particular, their draft status, old fixture-generation
proposal and source-porting language do not override the clean-room rules in
AGENTS.md or the current sibling project logs.

## Upstream and attribution

KwikScript is based on Rescript by Wassim Gharbi and its contributors. The
project preserves the upstream Git history, license, and required notice. This
fork is independent and is not an official Rescript release.

Upstream: <https://github.com/wassgha/rescript>

## License

Copyright (c) 2026 Wassim Gharbi and Rescript contributors.

Required Notice: Copyright (c) 2026 Wassim Gharbi and Rescript contributors
(https://github.com/wassgha/rescript)

Licensed under the [PolyForm Noncommercial License 1.0.0](LICENSE). The license
permits distribution of modified copies for permitted noncommercial purposes
when the terms and required notice travel with the software. It does not grant
general commercial-use rights. Resolve commercial licensing questions with the
upstream licensor rather than relying on this README as legal advice.
