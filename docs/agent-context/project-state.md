# Project State & Environment

## Machine / environment quirks (cost real time — read carefully)

### Node 25 breaks the test suite on develop-based branches

The user's machine runs **Node v25** (repo engines: `>=20`). Node ≥23 ships an experimental
Web Storage whose global `localStorage` shadows jsdom's in vitest and is non-functional
without a Node flag → `src/js/utils/localStorage.test.ts` fails (11 tests,
`localStorage.clear is not a function`) on any branch that lacks PR #51's fix.

- **PR #51's `vitest.setup.ts` installs a conditional in-memory Storage polyfill** — on
  branches containing it, tests just work.
- **On plain develop-based branches**, run tests with:
  `NODE_OPTIONS=--no-experimental-webstorage npx vitest run --silent`
- **Husky pre-commit runs the FULL test suite** (plus lint-staged, typecheck, knip) — on
  branches without the fix it blocks ALL commits. Use `git commit --no-verify` **only after**
  manually running `npm run check` and the test suite with the NODE_OPTIONS workaround.
  Do not use `--no-verify` to skip failing checks you haven't run.

### Dev server

- The user runs the app at **`http://localhost:4000`** (port via CLI flag; no `.env` file
  exists — `.env.sample` documents variables).
- Start it: `VITE_ENV=local VITE_VERSION=dev VITE_DATE=$(date +%s) npx vite --port 4000 --strictPort`
  — **`VITE_ENV=local` matters**: it enables the `window.__playerX` debug hook
  (`player.ts` bottom) used for live verification.
- **Do NOT pipe the background vite process through `head`/`tail`** — when the pipe closes,
  vite dies on SIGPIPE at the next HMR log. Redirect to a file instead.
- The user is logged into their Plex server (`dexplex`, plex.direct secure connection) in
  the app; session persists via localStorage.

### Browser verification (claude-in-chrome)

Techniques that worked well this session:

- `window.__playerX.getActivePlayer()` / `.getCurrentProgress()` — and on the gapless
  branch `.getGaplessState()` (playbackType HTML5|WEBAUDIO, decode state, engine window).
- **Import app modules directly through the Vite dev server** from the console:
  `const playerX = await import('/src/js/services/player.ts')` — full access to the real
  singletons (used to seek precisely when synthetic React range-input events proved flaky).
- Console-log filtering by the store's colour-coded logs (`--- playerSeamAdvance ---` etc.)
  is the cleanest way to observe effect flow. Console/network tracking starts only when
  first read — reload the page after attaching to capture boot logs.
- Chrome cast state: `cast.framework.CastContext.getInstance().getCastState()`;
  `NO_DEVICES_AVAILABLE` on macOS 15+ usually means Chrome lacks **System Settings →
  Privacy & Security → Local Network** permission (Spotify working proves nothing — it has
  its own permission and cloud discovery). This cost an hour; it's now in
  `docs/CHROMECAST.md` troubleshooting.
- Playback started for testing = real audio on the user's speakers. Pause when done.

### Git / GitHub

- Remotes: `origin` = adamdexter fork (push), `upstream` = chromatix-app (PR target).
- `gh` CLI authenticated as `adamdexter`. PRs: `gh pr create --repo chromatix-app/chromatix-app
--base develop --head adamdexter:<branch> --body-file <file>`.
- Commit trailer convention used all session:
  `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`; PR bodies end with the
  Claude Code attribution line. The maintainer values AI transparency (README §8) — keep it.

## Maintainer relations

- Solo maintainer (Reddit r/chromatix, Bluesky @chromaticnova.com). README §10.2 asks
  contributors to **check in before starting big features** — this session submitted three
  PRs without asking (user's call); a drafted Reddit post owning that is in the user's
  hands. Expect questions rather than instant merges; **#52 (gapless) is the one most
  likely to need discussion** (new dependency, wishlist item they called hard).
- Their wishlist (README §10.1): 1. player/gapless ✅ PR'd · 2. performance (virtualised
  lists, Plex API calls) ⬜ untouched · 3. automated testing ✅ PR'd.
- If the maintainer requests changes: the user will paste comments; each PR branch is
  self-contained and all have full local verification recipes in their feature docs here.

## Verification recipes (per branch)

```bash
# Full local gate (any branch):
npm run check          # knip + eslint + prettier + typecheck
NODE_OPTIONS=--no-experimental-webstorage npx vitest run --silent
npm run build

# E2E (Playwright) requires the maintainer's authed env + snapshots — NOT runnable here.
# PR #51's CI (.github/workflows/ci.yml) runs the same gate on Node 20/22/24.
```

## Session artifacts outside the repo

- **Claude memory** (`~/.claude/projects/-Users-adamdexter-GitHub-chromatix-app/memory/`):
  index + fork-setup, upstream-PRs, chromecast, multi-library notes. Kept in sync with
  these docs; these docs are the deeper source of truth.
- **Scratchpad** (session-temporary, may be gone): PR body drafts (`PR_DESCRIPTION.md`,
  `PR_TESTING.md`, `PR_GAPLESS.md`), Reddit post draft (`PR_REDDIT_POST.md`), issue drafts.
  The as-posted texts live in the GitHub PRs/issues themselves — fetch with `gh pr view`.
