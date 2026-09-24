# musicscheduler

Schedule music by time of day.

You already know you want something upbeat at 9am, R&B at 10, and something livelier
again at noon. Today that means walking over to a phone and changing it by hand, several
times a day, every day. This schedules those changes instead.

**Status:** pre-implementation. The product requirements are written; no application code
exists yet. See [PRD.md](PRD.md).

## How it works

You build a repeating day plan out of time slots, each pointing at a playlist you chose.
While you are listening, the schedule switches you to the right playlist as each slot
begins.

Some deliberate choices:

- **It never starts music on its own.** It redirects playback that is already happening.
- **It never cuts off a song.** A slot boundary waits for the current track to finish.
- **It does not care what you listen on.** Phone, laptop, Bluetooth speaker, wired
  headphones, or a smart speaker — it targets whatever Spotify session is currently
  active, with no device to pair or configure.
- **A vibe is a playlist you picked.** Nothing is inferred or auto-generated on your
  behalf.

## Requirements

- **Spotify Premium.** Spotify's API refuses playback control for free accounts.
- **Your own Spotify application registration.** This is self-hosted: you register an app,
  allowlist the accounts that will use your instance, and supply the credentials. Spotify
  caps each registration at five users and does not lift that for personal projects, which
  is why there is no central service to sign up for.
- **Somewhere to run it.** A small always-on container and a Postgres database. The
  schedule has to fire whether or not a browser is open, so it cannot run in the page.

Full setup is in [PRD.md](PRD.md), section 10.

## Other providers

Spotify only, and likely to stay that way. Apple Music and Tidal have no server-side
remote playback control at all — their APIs can only play audio inside an app running on
the listener's own device. Supporting either means a different kind of application, not
a different adapter. The design keeps that seam visible without promising it.
