# Agent Context — Chromatix Fork

> **Audience:** future AI agent sessions (and humans) continuing work on this fork.
> **Branch:** these docs live on the `agent-context` branch (off `develop`), deliberately
> outside the feature branches so PRs stay clean. Read them from any branch with
> `git show agent-context:docs/agent-context/<file>`.
> **Written:** 2026-07-14, at the end of the session that produced PRs #50–#52 and
> issues #53–#54.

## Current state snapshot (verify before trusting — see "First actions")

| Item                             | State                                                                                                 |
| -------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Fork                             | `adamdexter/chromatix-app` (origin) ← `chromatix-app/chromatix-app` (upstream)                        |
| Base branch                      | `develop` @ `d1b66fe` at time of writing                                                              |
| **PR #50**                       | Chromecast casting — `feature/chromecast` — OPEN, awaiting maintainer                                 |
| **PR #51**                       | Testing + CI + Node 23+ fix — `feature/automated-testing` — OPEN                                      |
| **PR #52**                       | True gapless playback — `feature/gapless-playback` — OPEN                                             |
| **Issue #53**                    | Player cache-key mismatch (currentLibrary vs track's library) — filed, fix PR offered after #50/#52   |
| **Issue #54**                    | Plex DASH uses account token not per-server accessToken — filed, fix PR offered after #50/#52         |
| Multi-library/server merged view | Researched, NOT built (user's choice) — full plan in `multi-library-plan.md` (repo root, this branch) |
| Reddit intro post                | Drafted for r/chromatix introducing the 3 PRs — posting status unknown (user's call)                  |

## Reading order

1. **`project-state.md`** — environment quirks (Node 25!), commands that work, verification
   techniques, maintainer relations. Read this FIRST; it prevents repeated mistakes.
2. **`codebase-learnings.md`** — architecture knowledge and repo conventions learned the
   hard way. Read before touching code.
3. **`feature-chromecast.md` / `feature-testing-ci.md` / `feature-gapless.md`** — deep
   context per PR: design decisions, rationale, verification, known follow-ups. Read the
   one relevant to your task.
4. **`bugs-and-followups.md`** — filed issues, suspected bugs not yet filed, and the queue
   of offered/likely follow-up work.
5. **`../../multi-library-plan.md`** (repo root) — the complete multi-library/multi-server
   feasibility research and phased plan. Self-contained; do not re-explore that topic.

## First actions for a new session

```bash
# 1. Where am I, what changed since these docs were written?
git -C /Users/adamdexter/GitHub/chromatix-app/chromatix-app branch --show-current
gh pr list --repo chromatix-app/chromatix-app --author adamdexter
gh issue list --repo chromatix-app/chromatix-app --author adamdexter
gh pr view 50 --repo chromatix-app/chromatix-app --comments   # …and 51, 52

# 2. If any PR has maintainer feedback, that is almost certainly the task.
# 3. Never commit without reading project-state.md § Husky/Node first.
```

## The prime directives of this fork

1. **Every contribution is a candidate upstream PR** to a cautious solo maintainer
   (README §10: check in before big features; values AI transparency — README §8). Design
   for mergeability: opt-in, additive, documented, tested, zero behaviour change by default.
2. **One branch per contribution, always off `develop`**, never stacked.
3. **Verify end-to-end before claiming done** — this session's standard: unit tests + full
   checks + live browser verification against the user's real Plex server, then the user's
   own hardware/listening test before any PR is submitted.
4. **The user's quality bar is high and explicit** — e.g. they rejected near-gapless:
   "if it's not true gapless, it's not worth shipping." Don't ship approximations of the
   thing; ship the thing or explain why not.
