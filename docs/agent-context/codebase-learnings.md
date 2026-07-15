# Codebase Learnings

Hard-won architectural knowledge and conventions. `.github/copilot-instructions.md` is the
maintainer's own conventions doc — read it too; this file covers what it doesn't say.

## Architecture (as of develop @ d1b66fe)

### The player stack (most-touched area this session)

- **Router pattern:** `src/js/services/player.ts` routes between sub-players that all share
  one interface (`init/unload/loadTrack/pause/resume/restart/setProgress/getCurrentProgress/
setVolume`). Sub-players: `player.native.ts` (HTMLAudioElement), `player.dash.ts`
  (dash.js/MSE for Plex transcodes); PR #50 adds `player.cast.ts`, PR #52 adds
  `player.gapless.ts`. **New playback capabilities should be new sub-players, not router
  restructures** — the router's `activePlayer` gating of callbacks (only the active player's
  events forward) is load-bearing and pinned by tests + e2e.
- **Critical units contract (pinned by tests):** `loadTrack(track, progressMs, play)` and
  `setProgress(progressMs)` take MILLISECONDS; `getCurrentProgress()` returns SECONDS;
  volume is 0–100 (sub-players divide by 100). `loadTrack` returns `false` only for the
  Plex-DASH-credentials-missing path → store sets `playerTrackError`, Resume retries.
- **All playback state logic lives in the store** (`models.player.js` effects), not in the
  services. Services translate engine events into the store's callbacks
  (`onLoadStart/onCanPlay/onEnded/onError`); `onEnded → playerNext(true)` is the ONLY
  auto-advance path.
- `usePlayerProgress` polls `getCurrentProgress()` on a 1s interval; store progress updates
  every 5th tick; scrubber resets on `realIndex` change.
- Track `src`/`thumb` URLs are fully baked (host + token) at transpose time. Plex DASH
  manifests are the exception — rebuilt at play time from globals (see issue #54).

### Store (Rematch)

- Models: `appModel` (auth, servers, content caches), `sessionModel` (user prefs + current
  server/library + play queue snapshot; persisted per-user to localStorage), `playerModel`
  (playback state/effects), `dialogModel`, `persistentModel`.
- Effects receive `(payload, rootState)` — `rootState` is a snapshot at invocation; re-read
  via `store.getState()` inside async continuations if staleness matters.
- Play queue: `playingTrackList` (original order) + `playingTrackKeys` (permutation) +
  `playingTrackIndex` (position in keys). Current track =
  `list[keys[index]]`. Shuffle recomputes keys keeping the current real track.
- `bridge.js` is the service-agnostic API layer; it reads 4 globals in a repeated preamble
  (~30 functions) — the tools beneath (`plexTools`/`jellyTools`) are stateless and fully
  parametrized. (Full map: `multi-library-plan.md` §4.)

## Conventions that matter for PRs

- **Tests:** colocated `.test.ts`, vitest `globals: true` (never import describe/vi),
  `describe('Testing "x" ...')` naming, fixtures in colocated `__fixtures__/`, browser-API
  mocks in root `__mocks__/`. Existing AI-assisted tests are headed
  `// Generated using GitHub Copilot`; this session's are headed
  `// Generated using Claude Code` (deliberate, transparency-consistent — flagged in PR #51).
- **tsconfig has a CLOSED `types` array** (`vitest/globals`, `vite/client`) — @types
  packages are NOT auto-included. Prefer minimal in-repo ambient `.d.ts` files
  (`src/types/cast.d.ts` is the precedent) over new @types devDependencies.
- **knip runs in the pre-commit gate** — unused exports fail the build. Don't export
  "just in case" helpers; knip.json sets `exports: off` only for some paths — check before
  assuming.
- **Icons:** raw SVG goes in `icons/general-original/`, compressed generated via
  `npm run svg:compress:new` (never hand-author the compressed copy — review caught this).
  Stroke-based (fill=none, `vector-effect="non-scaling-stroke"`), registered in `Icon.jsx`
  imports + `generalIcons` map alphabetically.
- **SettingsList** drives settings checkboxes by writing `sessionModel.setSessionState`
  directly; PR #52 added optional per-item `onChange` for settings that need an effect
  (dispatching a model effect instead).
- **Store console-log convention:** effects log `%c--- effectName ---` with a per-module
  colour (store #5c16b1, native #4c3ad4, dash #2f67d0, cast #e5a00d, gapless #1d9a6c).
  These logs are the primary live-debugging surface — keep the pattern.
- **Prettier runs on md/scss/json too** — always `npx prettier --check` new docs before
  committing (tables get realigned).
- **E2E snapshots:** `tests/pages/*.spec.ts` are visual-regression tests with golden
  screenshots needing the maintainer's authed environment. Any UI change to a
  snapshot-covered page (e.g. `/settings/controls`) deterministically fails them —
  note `npm run test:e2e:update:*` in the PR body for the maintainer (done in PR #50).
- The e2e player spec asserts `window.__playerX.getActivePlayer()` returns `'native'|'dash'`
  per codec — new sub-players must be OFF by default in test environments (cast: SDK absent
  in vanilla Chromium; gapless: setting defaults off + jsdom lacks AudioContext).

## Reusable session techniques

- **Multi-agent pattern that worked:** research fan-out (codebase sweep + external docs
  verification) BEFORE implementing; single-writer implementation (core code is too
  interdependent to fan out); parallel test-suite authoring (independent files only);
  adversarial review (N finders → dedup → 3 diverse-lens refuters per finding, ≥2 votes to
  confirm) AFTER committing. The cast review confirmed 7/14 findings — every confirmed one
  was real; the refuted half would have been wasted work.
- **Workflow-verified external facts** are recorded in `docs/CHROMECAST.md` and
  `docs/GAPLESS.md` (on their PR branches) — SDK URLs, codec matrices, CORS rules,
  transcoder params verified against official docs + real clients in July 2026. Trust but
  re-verify if much time has passed.
