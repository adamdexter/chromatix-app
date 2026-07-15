# PR #52 — True Gapless Playback (`feature/gapless-playback`)

**Status at writing:** OPEN. Verified by the user's ear with _The Dark Side of the Moon_
("flawless") plus live mechanical verification. User-facing/architecture docs:
`docs/GAPLESS.md` **on that branch**; this file adds agent context and the research
distillation.

## The decision history (matters for future work)

1. Research compared 5 architectures: dual-`<audio>` timed handoff (never sample-accurate,
   10-50ms best case), pure Web Audio (no streaming start, ~85MB PCM per 4-min track),
   hybrid HTML5+WebAudio, custom MSE SourceBuffer appends (YouTube Music's approach — needs
   demuxers/remuxers + gapless-tag parsers per browser; wrong for a solo-maintainer repo),
   WebCodecs (research project).
2. The repo contains the maintainer's own ARCHIVED dual-element prototype
   (`src/js/services/_archived/player.native.preload.ts` + test) and commented-out
   preloading stubs (`player.ts`, `models.player.js` `updateNextTrack`,
   `usePlayerProgress`). Plan A was to finish that design (near-gapless).
3. **The user rejected near-gapless explicitly**: "if it's not true gapless, it's not worth
   shipping." Pivoted to the hybrid architecture via **gapless.js**
   (RelistenNet/gapless.js, npm `gapless` v4.x, MIT, xstate dep, production on
   relisten.net). Key API facts: `Queue` with `gotoTrack/addTrack/removeTrack/seek/
setVolume`, callbacks `onStartNewTrack/onEnded/onError/onPlayBlocked`, `playbackMethod:
'HYBRID'`, `preloadNumTracks`, sample-accurate scheduling via `playbackEndContextTime`,
   built-in MediaSession position updates.

## Design in one paragraph

`player.gapless.ts` is a third sub-player. The STORE stays the queue's source of truth: the
module keeps a cheap queue mirror (`syncQueue(entries, {repeatOnce, repeatAll})`, entries in
`playingTrackKeys` order) and feeds the engine a sliding WINDOW (current + WINDOW_AHEAD=2;
`preloadNumTracks=1` to bound decoded-PCM memory). `nextQueueIndex()` mirrors
`playerNext(true)` semantics exactly — repeat-one appends the SAME src again (gapless
repeat-one works), repeat-all wraps, and the chain STOPS at transcode-needed entries (their
seam goes through the normal `onEnded → playerNext` flow, landing on dash/native). When the
engine crosses a seam itself, `onStartNewTrack` (filtered against `expectedEngineIndex` to
ignore echoes of our own actions, suppressed during rebuilds) fires
`onGaplessSeamAdvance({index})` → store effect `playerSeamAdvance` advances
`playingTrackIndex`, logs playback to the server, honours `disableRepeatOnceOnTrackChange`,
and re-syncs the mirror (which extends the engine tail via `reconcileTail`). Explicit user
loads rebuild the window; a re-request of the current track reuses it (seek/play). Off by
default behind Settings → Playback (replaced the maintainer's "not currently supported"
notice); iOS excluded (`isSupported()`: AudioContext + OS !== iOS).

## Sharp edges a future agent must not break

- **Echo suppression:** `expectedEngineIndex` + `suppress` flag are what distinguish "engine
  crossed a seam" from "we just called gotoTrack". Removing either double-advances.
- **`reconcileTail` never touches the currently playing engine entry** — only tail entries
  after `expectedEngineIndex` are removed/appended (repeat/shuffle toggles mid-play).
- **Toggle migration:** `playerGaplessToggle` reloads the current track at position through
  `playerLoadIndex` so engines swap mid-play; `setGaplessEnabled(false)` destroys the queue.
- The engine's `onEnded` covers BOTH real queue end AND chain-break-at-transcode — both are
  correct to forward to `params.onEnded`.
- `getDebugState()` (exposed via `__playerX.getGaplessState()` in local env) is the live
  verification surface: `playbackType` HTML5→WEBAUDIO crossover, `webAudioLoadingState`,
  `engineWindow`.
- Tests: `player.gapless.test.ts` (43) mocks the `gapless` package with a `vi.hoisted` fake
  Queue capturing constructor options + callbacks; `models.player.gapless.test.ts` (18)
  drives the real store. Filenames chosen to NOT collide with PR #51's suites.

## Known limitations (documented in docs/GAPLESS.md §4 — do not re-litigate)

Memory (~21MB/min PCM, preload capped at current+1), iOS excluded, CORS needed for the
decode fetch (per-track HTML5 fallback when blocked — playback never breaks, only the
seam), rare crossover blip (inherent to the hybrid architecture), hi-res resampled to
context rate, transcodes never gapless (no architecture can — fresh lossy sessions).

## Verified live (2026-07-14, user's Plex server)

Toggle migrates mid-play → `HTML5`+`LOADING` → crossover to `WEBAUDIO` ~12s in → seeked to
11s-before-end → `playerSeamAdvance` fired at the boundary with NO loadTrack/rebuild logs →
scrobble for the new track hit Plex (`/:/timeline` with new duration) → reload rejoined
paused at the right index. Then the user's listening test on DSOTM confirmed audible
seamlessness.

## Follow-ups / interplay

- Rebase vs PR #50: both add a sub-player to the same router switches; conflicts trivial
  but real. Whichever lands second rebases (offered to maintainer).
- Multi-server future: queue-mirror entries carry baked `src` (server-agnostic ✅); only the
  DASH-preload seam would need owning-server `withDashSrc` (multi-library-plan.md §5 P3).
- Possible future enhancement the research validated but wasn't built: short crossfade
  option to mask badly-tagged MP3 boundary noise (Gapless-5 has it; gapless.js doesn't).
