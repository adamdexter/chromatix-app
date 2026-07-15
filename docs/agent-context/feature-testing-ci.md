# PR #51 — Automated Testing + CI + Node 23+ Fix (`feature/automated-testing`)

**Status at writing:** OPEN. Zero production source touched — tests, fixtures, CI workflow,
and `vitest.setup.ts` only. Addresses README wishlist item 3.

## What's on the branch

1. **`vitest.setup.ts` storage fix** — Node ≥23's experimental Web Storage shadows jsdom's
   `localStorage`/`sessionStorage` in the vitest env (in vitest's jsdom, `window ===
globalThis`, and Node's broken accessor wins — `clear()` missing). Fix: a conditional
   in-memory `MemoryStorage` installed ONLY when `globalThis[key]?.clear` isn't a function;
   Node 20/22 keep jsdom's real storage. This also un-blocks the user's husky pre-commit.
2. **147 new tests** (suite 274 → 421):
   - `src/js/services/player.test.ts` (21) — router routing/gating. Pattern: `vi.hoisted()`
     mock factories for both sub-players so instances survive `vi.resetModules()`; re-import
     per test to reset module-level `activePlayer`; canPlayType spied on the jsdom audio
     prototype (requiresTranscoding has a module-level cache — after resetModules a FRESH
     instance exists; spy before importing).
   - `src/js/store/models.player.test.ts` (32) + `__fixtures__/playerSession.ts` — real
     Rematch store (`init({models})`), mocked boundaries only: `js/services/player`,
     `js/services/bridge`, `js/components` (PlaybackErrorMessage — the real barrel pulls
     JSX/SCSS and breaks), `js/utils` partial via `importOriginal` (only analyticsEvent
     stubbed). Covers next/prev/repeat/shuffle/mute state machine/loadTrack error contract.
     Header docblock documents deliberate scope exclusions (LOAD TRACKS family, error
     effects, refresh retry loop) — candidates for a future suite.
   - `plexTranspose.test.ts` (53) + `jellyTranspose.test.ts` (41) + fixtures — API
     normalisation contracts (appearance-artist logic, codec display maps, URL/token
     construction, jelly universal-vs-static src via canPlayType mock, thumb fallback tiers).
3. **`.github/workflows/ci.yml`** — first CI in the repo: knip/lint/prettier/typecheck/
   vitest/build on Node 20.x/22.x/24.x, PRs + pushes to develop/staging/production. E2E
   deliberately excluded (needs authed servers/snapshots).

## Suspected pre-existing bugs flagged in the PR body (not fixed, tests pin behaviour)

1. `models.player.js` `playerLoadIndex`: `bridge.logPlaybackPlay` + Play analytics fire even
   when `playerX.loadTrack()` returned false (failed load logged as playback).
2. `jellyTranspose.transposeLibraryArray` returns `undefined` for missing input (every
   other array transposer falls back to `[]`).
3. Jelly track links interpolate missing ids literally (`/artists/null`).

Fix PRs for these were offered in the PR body; the maintainer hasn't responded yet.

## Interplay with other branches

- The suites pin the develop player contract. PR #50/#52 add sub-players — their branches
  keep these tests passing because the new engines are inert by default (cast: no SDK in
  test env; gapless: setting off + no AudioContext in jsdom). **If #51 merges first, the
  feature branches need a trivial rebase; if a feature merges first, #51's
  models.player.test.ts mock factory may need the new playerX function names added**
  (closed vi.mock factories throw on missing functions the model now calls).
- The gapless branch has its own additional suites (61 tests) written to coexist
  file-wise with #51's (different filenames — `models.player.gapless.test.ts`).

## Review learnings (from the fresh-eyes audit before submission)

- Watch for: spies without scoped `mockRestore` (leak into later describes when only
  `clearAllMocks` runs), dead fixture imports (eslint has `no-unused-vars` OFF here!),
  unused fixture exports (knip exports check off for some paths), undocumented coverage
  gaps. All were found and fixed pre-submission; three shuffled-seed runs verified no
  order-dependence.
