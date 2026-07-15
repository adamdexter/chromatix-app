# Reddit post draft for r/chromatix

**Title:** Went down a rabbit hole and ended up with 3 PRs: Chromecast support, gapless playback, and a test suite

Hey! Long-time Chromatix user here — it's basically the only way I listen to my Plex library these days.

First, a mea culpa: I know the README says check in before starting anything big, and I… did not do that. What started as "I wonder if I can get this playing on my Chromecast Audios" turned into a proper rabbit hole, and I figured working code would be easier to evaluate than a proposal. Absolutely no hard feelings if any (or all) of these aren't the direction you want for the app — they were worth building just for my own setup.

The three PRs, in the order I'd suggest looking:

**#51 – Automated testing + CI.** Straight off your wishlist item 3. ~150 new unit tests around the player router, playback store logic, and the Plex/Jellyfin transpose layers, plus a GitHub Actions workflow running your existing husky checks on every PR. Also fixes the test suite on Node 23+ (Node's experimental localStorage shadows jsdom's and breaks the pre-commit hook). Zero production code touched.

**#50 – Chromecast support.** Casts to Cast devices with direct play for compatible codecs and server-side MP3 transcode for the rest (works for Plex and Jellyfin). Been running it daily on my Chromecast Audios around the house — Spotify Connect-style volume sync and everything. No new dependencies; docs included.

**#52 – Gapless playback.** Wishlist item 1, and yeah, you were right that it's the hard one. True sample-accurate gapless (not a timed-handoff fake) via gapless.js, opt-in beta toggle, off by default, with the trade-offs documented honestly. Tested with Dark Side of the Moon — seams are genuinely inaudible.

Full transparency in the spirit of your README's AI section: these were built working with Claude Code, with heavy review/verification along the way (the gapless PR includes the research on why the other approaches lose), and tested by me on real hardware.

Happy to rebase in whatever order suits you (#50 and #52 both touch player.ts), split anything up, or adjust the approach. And if you'd rather chat before reviewing, I'm around here or on GitHub (@adamdexter).

Either way — thanks for building Chromatix. 🎵
