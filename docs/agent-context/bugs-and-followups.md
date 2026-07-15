# Bugs, Filed Issues & Follow-up Queue

## Filed upstream issues (2026-07-14)

### Issue #54 — Plex DASH transcode URLs use the account token, not the per-server access token

- **The important one.** `withDashSrc` call sites pass `appModel.userToken` (plex.tv
  account/home-user token) into `getDashSrc`, while EVERY other server request uses
  `sessionModel.currentServer.accessToken` (verified on develop:
  `models.player.js:504-507,543-545` vs `bridge.js:~1700`).
- Suspected impact: transcode-needing codecs (ALAC/WMA/AIFF) fail on SHARED servers;
  owners unaffected (both tokens valid on owned servers) — which masks it. **Not
  reproduced** — inferred; stated honestly in the issue.
- Fix offered: pass `currentServer.accessToken` through `withDashSrc`/`getDashSrc` (+ the
  credential-wait in `playerRefreshTrack`). ~3 lines + tests. **Test with a Plex Home
  managed user before shipping** (attribution semantics) and ideally a real shared server.

### Issue #53 — Player builds track-cache keys from currentLibrary, not the track's library

- `playerLoad{Artist,Album,Playlist,Folder}` key `all*Tracks` caches by
  `currentLibrary.libraryId` while detail pages cache under the URL's libraryId
  (`models.player.js:334-336,385-387,425-427,465-467`). Repro: bookmark a detail page,
  switch current library, open bookmark, press play → key miss → redundant refetch; also
  makes recorded `playingLibraryId`/`playingServerId` untrustworthy.
- Low severity today (self-heals); fix = thread the item's real `libraryId` through the
  `playerLoad*` payloads (call sites: `ViewGrid.jsx`, `ArtistDetail.jsx`, `AlbumDetail.jsx`,
  `PlaylistDetail.jsx`), fallback to current.

**Agreed sequencing:** fix PR for #53+#54 together ("playback correctness"), but only AFTER
PRs #50/#52 resolve — all three touch `models.player.js`; a fourth concurrent diff there is
rebase hell for a solo maintainer. If the user says "fix the filed bugs", branch off
whatever develop looks like then.

## Known but NOT filed (deliberate)

- **Bare-int identity matching in rating/404 reducers** (`models.app.js:485-1107`):
  unreachable today (server switch wipes all state; ids unique within one server). It's a
  multi-server landmine only — documented in `multi-library-plan.md` §2/§3 as Phase 0 work.
  Filing it standalone would invite a fair "why?" from the maintainer.
- **Three transpose-layer oddities** flagged in PR #51's body (playback logged on failed
  load; `transposeLibraryArray` missing `|| []`; literal `null` in jelly links) — offered as
  follow-up PRs there; wait for maintainer signal.

## Follow-up queue (in rough priority order)

1. **Respond to maintainer feedback on #50/#51/#52** — highest priority whenever it exists
   (`gh pr view <n> --comments`). Rebase offers are on record for #50/#52 collision and for
   #51's mock factories vs new sub-players (see `feature-testing-ci.md` § interplay).
2. **Playback-correctness fix PR** (issues #53+#54) once #50/#52 settle.
3. **E2E snapshot regen note for #50** is on the maintainer (needs their authed env).
4. **README wishlist item 2 (untouched):** performance — virtualised List components
   (Tanstack Virtual) and Plex API call optimisation. No research done yet; start fresh
   with an exploration pass.
5. **Multi-library/multi-server merged view:** fully planned, not built —
   `multi-library-plan.md` (repo root, this branch). If green-lit: propose to the
   maintainer FIRST (it touches core state), then Phase 0. Do not re-explore.
6. Gapless extras if requested: crossfade option, member-library selection UI for the
   (future) merged view, Gapless-5-style features — see `feature-gapless.md`.

## Reusable multi-agent playbook (what worked this session)

1. **Research before code:** parallel workflow — one agent sweeps the codebase for
   integration points/contracts, one verifies external facts (SDK docs, API params, codec
   matrices) via web search. Feed both into the design. (Caught the wrong SDK URL, better
   Jellyfin params, and the maintainer's archived prototype before any code was written.)
2. **Implement single-writer.** Core changes are too interdependent for parallel edits.
3. **Fan out only independent files:** test suites (4 parallel authors, zero collisions),
   each agent required to get vitest+eslint+prettier+typecheck+knip green before returning.
4. **Adversarial review after committing:** N dimension-scoped finders → positional dedup →
   3 diverse-lens refuters per finding (code-trace / spec-check / severity-skeptic), ≥2
   isReal votes to confirm. Cast feature: 14 findings → 7 confirmed → all real, incl. two
   genuine races. Worth the tokens on anything touching playback.
5. **Live verification beats inference:** claude-in-chrome + `VITE_ENV=local` debug hooks +
   Vite-served module imports in the console. Then the USER verifies by ear/hardware before
   any PR is submitted. Every PR body states exactly what was verified and how.
