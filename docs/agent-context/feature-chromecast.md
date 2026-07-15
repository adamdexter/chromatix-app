# PR #50 — Chromecast Casting (`feature/chromecast`)

**Status at writing:** OPEN. Verified working on the user's Chromecast Audio units
(daily-driver feature for them). User-facing/architecture docs: `docs/CHROMECAST.md`
**on that branch** — read it first; this file adds the agent-relevant context that isn't
in it.

## Design in one paragraph

Third sub-player `player.cast.ts` wraps the Google Cast Web Sender (CAF) SDK with the
Default Media Receiver (`CC1AD845`, no Google registration). While a session is connected,
ALL playback routes to the cast device; local players stay idle. Chromecast-compatible
codecs (mp3/aac/flac≤96k24/vorbis/opus/wav — static allowlist in
`requiresCastTranscoding.ts`, NOT canPlayType-based) direct-play the baked `track.src`;
everything else gets a server-side progressive MP3-320 transcode URL (`getCastSrc.ts` Plex
universal transcoder with `protocol=http`; `getJellyCastSrc.ts` Jellyfin universal).
Progressive-only because adaptive (DASH/HLS) requires CORS on the media server, which
Plex/Jellyfin don't send; progressive needs none on the Default Media Receiver.

## Non-obvious decisions & facts (verified July 2026)

- SDK script: `https://www.gstatic.com/cv/js/sender/v1/cast_sender.js?loadCastFramework=1`,
  loaded lazily at init, skipped in Electron (no chrome.cast there — electron/electron#7024).
  `window.__onGCastApiAvailable` must be set BEFORE the script loads.
- Natural track end detection: `PLAYER_STATE_CHANGED` → IDLE + idleReason from
  `getCurrentSession().getMediaSession().idleReason` — `FINISHED` only (INTERRUPTED = our
  own next-load; CANCELLED = stop()). RemotePlayer has NO idleReason property.
- **Race fixed in review:** `intendedIdle` stays true from loadTrack until that load's
  `loadMedia()` promise resolves — a stale IDLE/FINISHED from the superseded track arriving
  in the ~100-300ms sender-notification window would otherwise double-advance the queue.
- Session lifecycle → store handoff: connect captures local progress BEFORE
  `syncCastRouting()` flips routing; disconnect resumes locally PAUSED at the receiver's
  `savedPlayerState.currentTime`; page reload auto-rejoins (ORIGIN_SCOPED) and ADOPTS remote
  state instead of reloading over it (guard in `playerRefreshTrack`:
  `castConnected && isCastMediaLoaded()`).
- Volume slider controls the DEVICE volume; on connect the app adopts the device volume
  (never blasts); `volumeRefresh` adopts rather than pushes while casting (review fix —
  boot-time push of persisted volume could blast speakers).
- Remote resume requires a queue to exist (`playerCastRemotePause` guard) — user-switch
  flow wipes the queue without unloading, and unguarded resume crashed `playerProgress`.
- `plex.direct` URLs resolve on Chromecasts because devices use Google DNS (8.8.8.8),
  bypassing router DNS-rebind protection — unless the router intercepts port 53
  (documented in troubleshooting).
- **macOS 15+ Local Network permission** gates Chrome's device discovery — the #1 support
  answer (verified live on the user's machine).

## Known follow-ups

- `/settings/controls` e2e golden screenshots need regeneration by the maintainer
  (`npm run test:e2e:update:settings`) — flagged in the PR body.
- If the multi-server work ever lands: `castSrc`/`getCastSrc` build from global creds —
  must move to owning-server resolution (see `multi-library-plan.md` §5 Phase 3). Cast
  receivers may also need a remote-capable connection from the server's ranked connection
  list rather than the browser's fastest (receiver network ≠ browser network path).
- Rebase note: touches `player.ts`, `models.player.js`, `types/player.ts` — conflicts with
  PR #52 are expected to be trivial (additive branches in the same router switch
  statements); whichever lands second rebases.

## Verification recipe

Unit: three utils suites on the branch. Live: Chrome + `VITE_ENV=local` dev server →
cast button appears when devices discovered (check `CastContext.getCastState()`), connect
→ `--- .cast - session started ---` log, handoff continues at position; volume sync both
directions; disconnect resumes locally paused. The user's real Chromecast Audios are the
ground truth.
