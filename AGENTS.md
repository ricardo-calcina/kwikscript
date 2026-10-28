# KwikScript - Agent Guidelines

Shared instructions for any AI coding agent (Codex reads this file directly; Claude Code reads it via `CLAUDE.md`).

## What this is

KwikScript is an Apple-Silicon fork of [Rescript](https://github.com/wassgha/rescript): a local, transcript-based editor for talking-head video. It cuts by transcript and silence, then exports FCPXML for Final Cut Pro. See `README.md` for features and scope.

Target: one reliable workflow on an M1 Mac. macOS 14+, Apple Silicon only, single-speaker English. Intel, Windows, Linux and the web build are not maintained. Don't add work for them unless asked.

## Git remotes

- `kwikscript` → `github.com/ricardo-calcina/kwikscript`, the fork. **Push here.**
- `origin` → `github.com/wassgha/rescript`, upstream. **Never push to `origin`.** Fetch from it only to pull upstream changes.

## Stack and layout

- npm (`package-lock.json`). Node 22, Homebrew FFmpeg at runtime.
- Next.js (App Router, client-only editor) packaged in Electron via static export.
- `app/`, `components/`: editor UI (transcript panel, timeline, cleanup sidebar)
- `lib/`: editor logic: store (zustand), edits, silences, fillers, alignment, timeline serialization, i18n
- `electron/`: main process, preload, menu, updater
- `native/coreml-transcriber/`: Swift helper (Parakeet v3 + word alignment via Core ML, FluidAudio pinned in `Package.resolved`)
- `scripts/`: build, asset copy, native build/benchmark, release notes
- `tests/`: i18n and timeline serialization tests
- `dist/`: local build output (gitignored)

## Checks

These match CI (`.github/workflows/ci.yml`):

```bash
npm run lint
npx next typegen && npx tsc --noEmit   # web typecheck
npm run typecheck:electron
npm run test:i18n
npm run test:timeline
npm run test:native:alignment          # native alignment, when touching lib/align* or native/
```

The dev machine is an M1 MacBook Air with limited headroom. Validate with the checks above. **Don't run `npm run build`, `npm run dist` or `npm run build:native:coreml` mid-task** unless the task is about building or packaging.

Run the app locally with `npm run electron:dev`.

## Docs and ownership

- `README.md`: operation, supported scope, checks, packaging and attribution.
- `LOG.md`: internal Now / Next / Decisions / Done. Keep active tasks in their
  owning repository; link sibling logs rather than maintaining duplicate lists.
- `DESIGN.md`: active design tokens and UI conventions, including existing edits.
- `docs/archive/`: historical plans, migration notes and benchmarks. Do not treat
  their status labels or proposed tasks as current instructions.
- `native/coreml-transcriber/README.md` and `patches/README.md`: component-specific
  operation and dependency patch context; keep them with the component.
- [N3 overview](../N3/README.md),
  [N3 log](../N3/LOG.md).

## Native boundary

The MIT native projects are separate from this licensed Electron fork. Native
implementation agents must not open or copy this fork's upstream code in `lib/`,
`components/`, `hooks/`, `electron/`, `workers/` or `app/` as reference. Use written
behaviour, hand-written fixtures and user-level comparisons. The owned Swift in
`native/coreml-transcriber/` and docs are allowed reference material. Historical
plans proposing fixtures generated from TypeScript are superseded.

Keep the installed Electron app/helper working while KwikLog's gym workflow
uses it. Do not remove it, retire it, or silently repoint consumers to KwikCore
as part of native migration or documentation work.

Release verification must distinguish local ad-hoc packages, Actions test
artifacts and a published/notarized release. The current tag/manual workflow
uploads unsigned test artifacts; the archived upstream release guide is not a
current publishing runbook. `release`, `cut:*` and `notes:preview` have external
side effects (publication/push/API use); do not run them as checks.

## License

Upstream's PolyForm Noncommercial license applies. Don't add anything that implies commercial distribution.
