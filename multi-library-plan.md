# Multi-Library / Multi-Server Merged View — Research & Implementation Plan

> **Status: RESEARCH COMPLETE, NOT BUILT.** This document is the full output of a deep
> feasibility investigation (July 2026). It is written agent-first: every claim carries a
> `file:line` reference verified against the codebase at the time of writing (branch
> `feature/gapless-playback`, based on `develop` @ `d1b66fe`), so a future implementation
> session can trust-but-verify rather than re-explore. Humans: read §1–§3 and §6; agents:
> everything, especially §4–§5.
>
> **Scope decided with the user:** merge multiple Plex libraries AND multiple Plex servers
> into one combined browsing view. Plex+Jellyfin cross-service merging is OUT of scope but
> must not be precluded by the design. UI model decided: virtual "All Music" entry, not
> multi-select (rationale in §3).

---

## 1. Feasibility verdict

**Feasible — no architectural rewrite required, but it is a multi-week refactor, not a
feature bolt-on.**

- Single-server multi-library subset: **~12–18 dev-days** (solo + AI assistance).
- Full multi-server vision: **~28–43 dev-days** (2–3 months part-time), dominated by the
  cache reshape, ~15 hook/page aggregations, and hardening (offline servers, stale tokens,
  restored queues).
- If the subset is built on the Phase-0 identity/route contract (§5), 100% of it carries
  forward to multi-server with no rework.

### Why it's cheaper than it looks

1. **The tools layer is already stateless.** Every function in `plexTools.js` /
   `jellyTools.js` takes explicit `{accessToken, serverBaseUrl, libraryId}` params
   (e.g. `plexTools.js:539` getAllLibraries, `:571` getAllArtists). Zero changes needed.
2. **Per-server tokens already exist.** Each server object carries its own `accessToken`
   from the plex.tv resource (`plexTranspose.js:82`); only the "current" one is ever used.
3. **Detail fetches are already parametrized.** `getArtistDetails(libraryId, artistId)`,
   `getAllArtistAlbums`, `getPlaylistDetails` etc. take `libraryId` from the caller (which
   reads it from `useParams()`, e.g. `pages/ArtistDetail.jsx:27`). Only **~10 array-level
   functions + search** read `currentLibrary` from the store and need fan-out:
   `getAllArtists`, `getAllAlbumArtists`, `getAllAlbums`, `getAllPlaylists`,
   `getAllCollections`, `getAllTags` (genres/moods/styles/tags), `getFolderItems`,
   `searchLibrary` (all in `bridge.js`).
4. **Caches are already a flat multi-library pool.** Content maps are keyed
   `${libraryId}-${id}` (`models.app.js:581,597,613,629,794,950,1023,1123,1160`); hooks
   merely FILTER by `currentLibraryId` (`useGetArtistArray.js:53-55`, `useGetAlbumArray.js:57-59`,
   `useGetPlaylistArray.js:56-58`). Merging is largely "relax the filter".
5. **Track/artwork URLs are baked at transpose time** with host+token embedded
   (`plexTranspose.js:448` src, `:21-28` getThumb) — portable regardless of "current" server.
6. **The maintainer anticipates this.** An `allAccounts` multi-account registry refactor is
   pre-sketched in comments at `models.app.js:55-72`.
7. **Latent hooks exist.** `sessionModel.playingServerId`/`playingLibraryId` are WRITTEN on
   every load (`models.player.js:361,401,441,481`) but never read anywhere — ready-made for
   playback ownership routing.

### The real work

1. **Identity gap.** No item, cache key, link, or route carries `serverId`. Plex
   `libraryId` is a small-int section key (commonly `"1"`) and item ids are per-server
   ratingKey ints — BOTH collide across two Plex servers. (Jellyfin uses GUIDs — safe.)
2. **Single global `serverBaseUrl`** (`models.app.js:80`), written by one
   fastest-connection race (`bridge.js:290-309`). Two concurrent fetches against different
   servers would cross-wire tokens/hosts into baked URLs nondeterministically — a per-server
   connection registry is a prerequisite for ANY concurrent multi-server fetching.
3. **~30-function bridge preamble.** Nearly every `bridge.js` function re-reads 4 globals
   (`currentServer.accessToken`, `currentService`, `serverBaseUrl`,
   `currentLibrary.libraryId`) — e.g. `bridge.js:376-379`, `:460-463`, `:926-931`,
   `:1568-1572`. Mechanical to centralize, broad diff.
4. **Playback-adjacent credential use is global.** Plex DASH manifests are rebuilt at play
   time from the GLOBAL `serverBaseUrl`+`userToken` (`models.player.js` `withDashSrc`
   ~:1004, `playerRefreshTrack` :111-139); scrobbling (`bridge.js:1697-1766`) and
   ratings/favourites (`bridge.js:1603-1672`) always hit the current server. A merged queue
   would silently scrobble/rate against the wrong server — where a colliding ratingKey may
   belong to a DIFFERENT track (silent data corruption on the user's server).
5. **Routing funnel.** Routes are `/libraries/:libraryId/...` (`routes.ts:59-241`);
   `BrowserRouteValidate.jsx:20-56` checks only PRESENCE of user/server/library; nothing
   validates or syncs the URL `libraryId` against `currentLibrary`; `isStoreReady()`
   (`bridge.js:33-37`) gates all calls on a single current pair.
6. **Fetch serialization.** Module-level `*Running` booleans in `bridge.js` serialize each
   call type globally — fan-out needs an in-flight promise map keyed per source.

---

## 2. Latent bugs discovered (exist TODAY, worth upstream issues regardless)

1. **Player cache-key mismatch for foreign-library deep links.** `playerLoadArtist/Album/
Playlist/Folder` build track-cache lookup keys from `sessionModel.currentLibrary.libraryId`
   (`models.player.js:334-336, 385-387, 425-427, 465-467`), but detail pages cache tracks
   under the URL's real libraryId. Navigating a deep link to a non-current library and
   pressing play → key mismatch → silent refetch/dispatch loop.
2. **Plex DASH uses the account `userToken`, not the per-server `accessToken`**
   (`withDashSrc` call sites `models.player.js:515,519,559-561`). Plausibly broken for
   transcoded playback on SHARED servers today. (Spike this — see §5 Phase 3.)
3. **Rating/404 reducers match by bare int id.** `setTrackRating`/`setAlbumRating`/
   `storeXxx404` etc. match caches via `findIndex(a => a.artistId === payload.artistId)`
   (`models.app.js:485-1107`). The moment two servers' data coexists in the store, rating
   server A's track 12345 can mutate server B's unrelated cached item 12345.

---

## 3. Decided design (with rationale)

### UI model: virtual "All Music" (rejected: multi-select checkboxes as primary UX)

- One pinned **"All Music"** row in `UserMenu.jsx` (the existing server+library switcher,
  `:81-135`); single-select semantics preserved.
- Route sentinel **`/libraries/all/...`** — cannot collide (Plex serverIds are 40-char hex,
  library ids ints, Jellyfin GUIDs).
- State: **a selection set in `sessionModel`**, NOT a synthetic entry in
  `appModel.allLibraries`:

  ```js
  librarySelection: {
    mode: 'single' | 'merged',
    members: [{ serverId, libraryId }],   // single mode: exactly one member
  }
  ```

  Rationale: `allLibraries` is fetched server data, torn down on switch and revalidated by
  `validateCurrentLibrary` (`models.session.js:791-822`) — a pseudo-entry would special-case
  every consumer. A merged view is a user preference, which is what `sessionModel`
  persists per-user already.

- **Detail pages always use real `(serverId, libraryId)`** — items keep real baked links, so
  click-through from a merged grid reuses existing single-library code paths forever. Merge
  complexity is confined to array views + search.
- **Duplicates** (same album on two servers): show both with a server badge. Dedupe ONLY in
  display selectors, NEVER in the store (a hidden copy's ratings/playability would become
  unreachable, irrecoverably). An optional "collapse duplicates" toggle can group at the
  selector layer later.
- Member-selection checkboxes = a later small settings surface; v1 semantics: "All Music" =
  all music libraries.
- Sidebar: link prefix from a `useSourcePathPrefix()` helper; hide Folders under the merged
  view (folder trees don't merge meaningfully) — `SideBar.jsx:31,130-368`.
- View/sort settings stay global per-user (as today, `models.session.js:158-244`).

### Identity scheme

- **`serverId` = Plex `clientIdentifier`** (from plex.tv resources, `plexTranspose.js:78-85`) —
  stable across IPs/reboots; what app.plex.tv itself uses in URLs. Jellyfin `server.Id`
  (`jellyTranspose.js:118`) has the same properties.
- **Stamp `serverId` on every transposed item** at transpose time (thread the server through
  transpose entry points).
- Cache/route keys: `sourceKey = ${serverId}:${libraryId}`, `itemKey = ${sourceKey}:${itemId}`
  (`:` is delimiter-safe for all id alphabets involved). Rejected: registry-index sourceIds
  (session-local, break deep links/persisted queues) and slugs (nothing slug-stable exists).
- **Route shape: `/libraries/:serverId/:libraryId/...`** with `/libraries/all/...` for the
  merged view. Legacy `/libraries/:libraryId/...` gets one generation of redirects resolving
  `:serverId` from the current server (precedent + machinery: `routes.ts:340-537` legacy
  table, `xRedirect` interpolation in `BrowserRouteSwitch.jsx:48-62`). React Router v5
  distinguishes arity, so both patterns coexist during transition. Bump `playingVersion`
  (`models.session.js:347`) once to bust persisted queues with old baked links.

### Connection registry (new Rematch model, e.g. `models.connections.js`)

```js
connections: { [serverId]: { status: 'idle'|'connecting'|'connected'|'failed',
                             baseUrl, error, checkedAt } }
```

- Reuse `plexTools.getFastestConnection({server})` (`plexTools.js:489-533`) per server —
  already a pure per-server function. Jellyfin = trivially-connected entry.
- `ensureConnected(serverId)` effect with in-flight promise memoization; lazy connect on
  first demand, parallel prefetch of all members when merged mode activates.
- Per-server failure is non-fatal in merged mode (error chip + retry via the existing
  `appModel.addNotification` channel); the global fatal-error modal remains only for
  single-source mode.
- Abort scoping: `plexTools.abortControllers` (`plexTools.js:172-180`) becomes a
  `Map<serverId, Set<AbortController>>`; `bridge.abortAllRequests(serverId?)`.
- Delete `appModel.serverBaseUrl`; derive from registry at call time. Persist ids only —
  never connection URLs (they go stale).

### Source-aware bridge

- One `resolveContext({serverId, libraryId})` helper replaces the 30-function preamble;
  omitting the source resolves to current — single-source callers keep working during
  migration.
- `*Running` booleans → in-flight promise map keyed `${fnName}:${sourceKey}` — enables
  parallel fan-out, coalesces duplicates, and fixes the silent-drop quirk where a second
  caller mid-fetch gets `undefined` (`models.player.js:339-343` re-dispatch loop).
- List caches reshape to per-source buckets with status (subsumes the global `haveGotAllX`
  booleans, kills the `isExtra` hack at `models.app.js:495-501`):

  ```js
  sources: { [sourceKey]: { artists: {status, items, error}, albums: {…}, … } }
  ```

  Detail caches keep their LRU-5 (`maxDataLength`, `models.app.js:12`) — only key format grows.

### The 5 corner-painting decisions (get these right in the FIRST phase)

1. **URL scheme gains `:serverId` before any merged view ships.** Links are baked into
   cached items AND localStorage-persisted queues; shipping merged views on the old shape
   buys a second transpose migration + second `playingVersion` bust.
2. **`(serverId, id)` identity matching in reducers from the moment two servers' data can
   coexist** — not when merged views ship (§2 bug 3). Sessions store ids, not stale server
   objects.
3. **Never dedupe/merge at fetch/store time** — display selectors only.
4. **Kill the global `serverBaseUrl` before any concurrent multi-server fetching** — and
   land abort scoping with it.
5. **All playback-adjacent credential use goes through owning-server resolution** — and
   in-flight PRs (#50 cast `castSrc`, #52 gapless preload via `withDashSrc`) should be
   rebased onto that helper rather than adding more global-cred call sites.

---

## 4. Verified architecture map (for the implementing agent)

### Load-bearing singletons

| State                                   | Where        | Ref                    |
| --------------------------------------- | ------------ | ---------------------- |
| `currentService` ('plex'\|'jellyfin')   | appModel     | `models.app.js:50`     |
| `userToken` (Plex account token)        | appModel     | `models.app.js:75`     |
| `serverBaseUrl` (single resolved URL)   | appModel     | `models.app.js:80`     |
| `allServers` / `allLibraries`           | appModel     | `models.app.js:76,81`  |
| `currentServer` (carries `accessToken`) | sessionModel | `models.session.js:25` |
| `currentLibrary`                        | sessionModel | `models.session.js:26` |

- Service singleton: one active adapter `serviceTools[currentService]` (`bridge.js:15-18`);
  single-token localStorage (`bridge.js:90-110`) — this is why cross-service merge needs the
  `allAccounts` refactor and is out of scope here.
- Server switch = full teardown: `switchCurrentServer` (`models.session.js:717-734`) aborts
  everything, wipes server+library state (`models.app.js:450-479`), unloads the player. This
  teardown is today's correctness safety net — and remains the P2 "switch-to-play" gate.
- Selection flows: `ViewServers.jsx:22-74` (full-page picker), `UserMenu.jsx:81-135`
  (switcher), auto-select single server/library (`models.session.js:736-767, 791-822`).
- Boot funnel: `BrowserRouteValidate.jsx:20-56` linear user→server→library redirects;
  `useGotRequiredData.ts` gates the same sequence.

### Data flow

- Pipeline: hook `useEffect` → `bridge.getX()` (reads 4 globals) →
  `serviceTools[service].getX({accessToken, serverBaseUrl, libraryId})` → `transpose*()` →
  `store.dispatch.appModel.storeX/setAppState`.
- Item shape: `kind`, `libraryId`, one id, `title`, `link='/libraries/{libraryId}/…'`,
  baked `thumbSm/thumbMd`; tracks add `src` (baked), `codec`, `trackKey` (Plex).
  **No `serverId` anywhere.**
- `refetchData = true` (`bridge.js:25`) → every mount refetches; important for fan-out cost.
- Search: `searchLibrary2` (`bridge.js:1561-1597`), monotonic `searchCounter` staleness
  guard; results transposed with current-library links (`plexTranspose.js:499-577`).

### Playback

- Router `player.ts` → native (baked src) / dash (rebuilt manifest) / gapless (PR #52
  engine; queue mirror entries carry baked `src` — already server-agnostic).
- `playerLoad*` effects read caches by `currentLibrary.libraryId + '-' + id`
  (`models.player.js:334-336` et al) — see §2 bug 1.
- Queue snapshot persisted per-user (`store.ts` saveSessionData); includes
  `playingServerId/LibraryId` (write-only today).

---

## 5. Phased implementation plan

> Each phase is independently shippable and upstream-mergeable. Phases 0+1 ≈ the
> single-server subset (~12–18 days). PR #50 (cast) and #52 (gapless) both touch
> `models.player.js`/`player.ts` — land playback-touching phases after they merge, or
> rebase on them.

### Phase 0 — identity + contracts groundwork (~8–12 days; no visible change)

- Stamp `serverId` on every transposed item; thread server through transpose entry points
  (`plexTranspose.js`, `jellyTranspose.js`).
- Central helpers (new util module): `buildLibraryPath()`, `parseLibraryParam()`,
  `getItemKey()` — ban ad-hoc `'/libraries/'+id` / `${libraryId}-${id}` concatenation.
- Route contract: `/libraries/:serverId/:libraryId/...` + legacy redirects; bump
  `playingVersion` once.
- Fix reducer identity matching to `(serverId, id)` (`models.app.js:485-1107`).
- Thread real `libraryId` through `playerLoad*` payloads (fixes §2 bug 1; call sites:
  `ViewGrid.jsx`, `ArtistDetail.jsx`, `AlbumDetail.jsx`, `PlaylistDetail.jsx`).
- Sessions persist ids, not server objects.

### Phase 1 — merged multi-library view on ONE server (~1–1.5 weeks)

- `librarySelection` in sessionModel; "All Music" row in `UserMenu.jsx`; hide Folders in
  `SideBar.jsx` when merged.
- Fan out the ~10 `getAll*` functions + `searchLibrary` over members
  (`Promise.allSettled`, concat, single dispatch keeps `haveGotAllX` semantics).
- Hook filters → `isItemInLibrary(item, selection)` (~8 `useGet*` hooks).
- Partial-failure UX: loaded sources render; failed ones surface a notification.

### Phase 2 — multi-server browse, switch-to-play gated (~1.5–2.5 weeks)

- Connection registry model + `ensureConnected` + per-server abort scoping.
- `resolveContext()` bridge refactor; in-flight promise map replaces `*Running` booleans.
- List caches → per-source buckets; detail cache keys → composite; `allLibrariesByServer`.
- UserMenu lists all servers' libraries; foreign-library click = existing teardown switch.
- Cross-server PLAY shows "Switch to {server}?" dialog (reuse `models.dialog.js`) —
  playback correctness never at risk while shipped.

### Phase 3 — seamless cross-server playback (~1 week)

- **Spike FIRST:** do shared-server transcode manifests accept the per-server
  `accessToken`? (Today they get the account `userToken` — §2 bug 2. This is the only
  unknown that could force a redesign.)
- Route `withDashSrc`, `playerRefreshTrack` credential-wait (read `playingServerId` as the
  reconnect hint), `logPlayback*`, `setStarRating`/`toggleFavourite` to the owning server.
- Remove the P2 gate. Verify mixed-server queues with gapless (#52 preload path) and cast
  (#50 `castSrc` must use owning-server resolution; cast receivers may need a
  remote-capable connection from the ranked list rather than the browser's fastest).

## 6. Verification plan

- **Unit:** extend PR #51's suites — transpose stamping, key/path helpers, `resolveContext`
  defaulting, `(serverId,id)` reducer matching, selection/fan-out store effects.
- **Live (two real Plex servers):** merged grids render both sources with correct art;
  direct + transcoded playback from each server; ratings/scrobbles land on the correct
  server (check each server's dashboard); kill one server mid-browse → other keeps working
  with error chip; deep links + legacy URLs redirect; restored queue reconnects to the
  owning server after reload.
- **Regression:** full unit + e2e suites; single-library users must see zero change
  (Phase 0 is a pure refactor — assert byte-identical behavior).

## 7. Related context

- Open upstream PRs from this fork: #50 (Chromecast), #51 (testing/CI — the suites to
  extend), #52 (gapless). All three predate this plan; #50/#52 interplay noted in §5.
- The user's environment note: Node 25 breaks `localStorage.test.ts` on branches without
  PR #51's `vitest.setup.ts` fix — run tests with `NODE_OPTIONS=--no-experimental-webstorage`.
- Maintainer relations: solo maintainer, asks contributors to check in before big features
  (README §10.2). This refactor touches the app's core state — **propose before building.**
  Phase 0 doubles as three latent-bug fixes (§2), which makes a strong opening conversation.
