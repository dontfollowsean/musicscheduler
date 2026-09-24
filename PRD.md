# Day-Scheduled Music ("vibe scheduler") — Product Requirements

> *Status: v1 · 2026-09-24 · Owner: Sean Wilkinson*
> *Platform claims in this document were verified against live vendor documentation on 2026-09-24. See §12 for sources.*

---

## 1. Problem

Music plays in the kitchen all day. What suits 9am does not suit 10am, and what suits 10am does not suit noon. Today the only way to change that is to walk over, pick up a phone, and choose something else — several times a day, every day, for a change that is entirely predictable.

The taste is not the problem. The person already knows they want upbeat at 9, R&B at 10, and something livelier at noon. **The problem is purely that a known, repeating intention requires manual execution.** Every other repeating daily intention — lights, thermostat, alarms — has been automated. Music has not, because streaming services expose playlists as things you pick, never as things you schedule.

**Who it is for.** Someone who plays music for hours at a stretch in a shared space — a kitchen, a workshop, a home office — who has already curated playlists for different parts of the day, and who is interrupted by having to switch between them. They have Spotify Premium. They may or may not own a smart speaker; the product must not care. They are willing to run their own instance (§10).

**Not for**: people building a party queue, people who want music discovery, people who listen in discrete 20-minute sessions. The unit of value is *the day*, not the session.

---

## 2. Platform constraints that shape the product

These are not background notes. Each one forces a rule, a non-goal, or a piece of the setup story, and they are recorded here so no requirement below looks arbitrary.

**C1 — Five authenticated users per Spotify app registration, and the cap cannot be lifted.** A newly created Spotify app is in *development mode*: "Up to 5 authenticated Spotify users can use an app that is in development mode," and "Each Spotify user who installs your app will need to be added to your app's allowlist before they can use it." Leaving development mode requires a quota extension, and as of 15 May 2025 Spotify "only accepts applications from organizations (not individuals)," with implementation requirements including a legally registered business entity, a launched service, and **at least 250,000 monthly active users**. Read in order, that list is circular: to be allowed more than five users you must already have a quarter of a million. There is no ladder, and a hobby or pre-launch project cannot qualify. A separate licensing wall closes the usual escape route — every Player endpoint carries "Streaming applications may not be commercial," so monetisation is prohibited too.

This is why the product is distributed as something you install rather than something you sign up for (§10), and why a single instance serves at most five accounts.

A trap worth coding for: "Users may be able to log into a development mode app without having been allowlisted by the developer. However, API requests with an access token associated to that user and app will receive a **403** status code error." OAuth *succeeds* for a non-allowlisted account and everything afterwards fails.

**C2 — Spotify Premium is required, unconditionally, for every playback write.** `PUT /me/player/play`, `POST /me/player/queue` and `PUT /me/player` all carry the identical sentence: "This API only works for users who have Spotify Premium." `GET /me/player` carries no such note — reads work for free accounts, writes do not. A free user could otherwise log in, build a whole day plan, and have it silently never fire. Additionally, "The app owner must have a Spotify Premium account for apps in development mode to function," so the person deploying the instance needs Premium as well.

**C3 — Playback targets whichever device is currently active, and no device is ever pinned.** On `PUT /me/player/play`'s `device_id` parameter: "The id of the device this command is targeting. **If not supplied, the user's currently active device is the target.**" The same sentence appears on `POST /me/player/queue`. The server therefore never needs to discover, store, or pin a device; it issues one call and Spotify routes it to the phone, desktop app, browser tab, or Connect speaker actually in use. Bluetooth and wired headphones are downstream of a phone or laptop, so they are supported for free.

The hard edge of that promise: there must *be* an active device. When nothing has played and the Spotify client has gone fully inactive, `GET /me/player` returns **204 No Content** and there is no device to address; a play call then fails. `PUT /me/player` (Transfer Playback) does require an explicit device, but this product has no reason to call it.

**C4 — Credentials expire on a hard six-month wall clock.** "Refresh tokens issued to apps registered in the Developer Dashboard have a lifetime of 6 months." The clock starts when the user authorizes the app; "Refreshing an access token does not extend the refresh token's lifetime"; after six months the token can no longer be used and the user must be sent through authorization again, which starts a new six-month lifetime. Inactivity is *not* the failure mode — an idle token and one refreshed a thousand times die on the same day. Access tokens remain one hour. The failure signal is `invalid_grant`, on which the app "should discard the refresh token and start the appropriate authorization code flow instead of retrying."

Because the scheduler is server-side and can hold a client secret, it uses plain Authorization Code rather than PKCE: PKCE rotates refresh tokens and would introduce rotation races, and rotation buys nothing when the six-month clock runs from authorization rather than last use.

**C5 — Recommendations, audio features, and Spotify-owned playlists are unavailable to new apps.** The November 2024 Web API change removed, for apps registered after that date and for apps still in development mode, access to: Related Artists, Recommendations, Audio Features, Audio Analysis, Get Featured Playlists, Get Category's Playlists, 30-second preview URLs in multi-get responses, and **algorithmic and Spotify-owned editorial playlists**. All the affected reference pages still carry `Deprecated` banners ~22 months later; nothing has been restored.

Two consequences, both load-bearing. Auto-generated or inferred vibes are not merely out of scope, they are **impossible**. And Discover Weekly, Daily Mix, Chill Hits, and anything from the Browse tab **cannot be used as vibes** — only playlists in the user's own library.

---

## 3. Product principles

**P1 — The day plan is the product.** Users author a repeating plan once and it runs every day. There is no "set up this listening session." If a user has to touch the app daily, the product has failed at the thing it exists to do.

**P2 — Never cut off a song.** A scheduled transition waits for the current track to end, then switches. A slot boundary is a statement about what should be playing *next*, not a command to interrupt. (One deliberate carve-out: TR-4.)

**P3 — A vibe is a playlist the user picked.** The product never infers mood, never auto-generates a mix, never decides on the user's behalf what "upbeat" means. A vibe is a named pointer to a playlist the user chose. This is both the intended design and the only one the platform permits (C5).

**P4 — The schedule has a memory.** If a slot boundary passes while nothing is playing, the plan does not forget. When the user next presses play, they land in whichever slot is active *at that moment* — not the one that was playing when they stopped.

**P5 — Never require a particular device.** Phone speaker, Bluetooth, wired headphones, laptop, smart speaker: all equal. The product targets whichever Spotify session is currently active and never pins a device (C3).

**P6 — Silence is honest.** When the product cannot act — connection expired, no active device, a playlist went away — it says so visibly. It never pretends a plan is running when it is not. Without this, C4 guarantees a silent failure at around six months.

---

## 4. Core model

Four objects.

**Vibe** — a user-facing name bound to one provider playlist.
`{ id, name, provider, provider_playlist_uri, playback_options }`
The name ("Morning lift", "RB jams") is the user's; the playlist is theirs too. Decoupling the two lets a user re-point "Morning lift" at a different playlist without touching any schedule. `playback_options` carries shuffle intent (see §5, "Running out of playlist").

**Slot** — a start time and the vibe that should be playing from then on.
`{ id, start_time (local wall clock), vibe_id }`
A slot has a start and **no end**: it runs until the next slot starts, or until the plan's end-of-day. Modelling only starts is deliberate — it makes gaps and overlaps structurally impossible, and it matches how people actually describe this ("R&B at 10").

**Day plan** — an ordered set of slots plus the days it applies to.
`{ id, name, timezone, days_of_week[], slots[], active }`
The timezone lives on the plan and slot times are local wall-clock, so the plan survives daylight-saving transitions the way an alarm clock does. *(Edge case: a 02:30 slot on a spring-forward day does not exist. Rule: a slot whose local time is skipped fires at the first valid instant after the jump.)*

**Connection** — an authorized provider account.
`{ id, provider, external_user_id, refresh_token (encrypted), authorized_at, expires_at, status }`
`expires_at` = `authorized_at + 6 months`, because that is a hard platform fact (C4), and it is stored explicitly so the product can warn ahead of it rather than discover it as an outage.

### The transition rules, stated normatively

**TR-1 (scheduled transition).** When a slot boundary is reached and playback is active, the system waits for the current track to complete and then starts the new slot's vibe. It **never** interrupts a track in progress. *(P2)*

**TR-2 (boundary accuracy).** The switch is issued after the current track has demonstrably ended, never before. The new vibe therefore begins approximately **1–2 seconds into whatever would have played next**. This is a deliberate choice: being slightly late cuts a track that has barely begun, while being slightly early clips the end of a song the user was listening to — which is precisely what P2 forbids. **Transitions are not gapless and must not be described as such.**

**TR-3 (deferred transition / catch-up).** If a slot boundary passes while nothing is playing, no action is taken and nothing is queued. When playback is next observed, the system determines which slot is active *at that observed moment* and transitions to it. It never resumes an earlier slot. *(P4)*

**TR-4 (catch-up carve-out).** When TR-3 fires and the observed track started less than **15 seconds** ago, the system switches **immediately** rather than waiting for it to finish. Rationale: P2 protects a song the user is *listening to*; a track auto-started by resume is not that, and honouring P2 literally here would mean up to four minutes of the wrong vibe — defeating P4 at the exact moment it was supposed to help. Above the 15-second threshold, TR-1 applies as normal.

**TR-5 (no cold start).** The system never originates playback from silence. It only redirects playback that is already happening. If there is no active device, the plan waits (C3).

**TR-6 (manual override wins the moment, not the day).** If the user manually changes what is playing, the system does not fight them: it does not revert. The next *scheduled* slot boundary still fires normally. Overriding is a statement about now, not a cancellation of the plan.

---

## 5. User experience

**Connect.** Sign in with Spotify against the instance's own Spotify application. Premium is verified at connect time (`GET /me` → `product`); a non-Premium account is refused with an explicit explanation rather than admitted to a product that will silently never work for them (C2). Because OAuth succeeds for accounts that are not on the app's allowlist and only later calls fail with 403, the app probes once immediately after callback and tells a non-allowlisted user to ask the instance's owner to add them, rather than showing a generic failure (C1).

**Build a plan.** Pick playlists from your own library (`GET /me/playlists`), name them as vibes, and place them at times on a day. Choose which days of the week the plan runs.

> **Surfacing a platform limit as product design.** Spotify-owned playlists — Discover Weekly, Daily Mix, Chill Hits, anything from the Browse tab — are **not accessible** to this app and cannot be used as vibes (C5). The picker must show only what will actually work, and where a user might reasonably look for an editorial playlist it should explain in one line why it isn't there and suggest saving a copy to their own library. Discovering this limit through a failed 9am transition is unacceptable.

**Watch it run.** A single honest status line: which slot is active, what the plan will do next, and — critically — whether the plan *can* run right now. Connection expiring in 9 days, no active device seen today, a vibe's playlist deleted: all visible here, per P6.

**Running out of playlist.** A slot can outlast its playlist: a 90-minute slot with a 40-minute playlist. When the playlist ends before the slot does, **Spotify's own autoplay continues past the end of the playlist**. The transition does not set repeat-context, and the product does not re-start the vibe.

Stated plainly, because it is a known and accepted consequence rather than an oversight: **over a long slot, autoplay drifts away from the playlist the user deliberately chose.** Autoplay picks its own continuation, so the last hour of a long slot may not sound like the vibe that was scheduled. That is accepted in exchange for music that keeps playing without looping a 40-minute playlist twice. The next slot boundary corrects the drift by switching to the next vibe's playlist as normal.

---

## 6. Why the scheduler runs on a server

**A browser cannot do this job**, and the reason is not performance:

1. **Nobody has the tab open at 9am.** A scheduled transition must fire with no client present. That alone settles it.
2. **A Web Playback SDK device dies with its tab**, so a browser-hosted player is also a device that disappears.
3. **Detecting "the user pressed play" requires continuous observation.** There is no webhook; something must poll, and it must be awake when the user is not looking at the app.
4. **Refresh tokens must live somewhere durable and secret** for six months (C4).

**What makes the server side tractable** is C3: `PUT /me/player/play` with `device_id` omitted targets whatever device is currently active. The server never needs to know what the user is listening on — phone, laptop, Bluetooth speaker, headphones. It issues one call and Spotify routes it. **This is why P5 costs nothing to honour, and it is why this design deliberately does not store or pin device IDs.** A future contributor tempted to "improve" reliability by pinning a device would be breaking P5, not strengthening it.

---

## 7. Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  Web app (Next.js / SvelteKit — whatever; it is a thin CRUD UI)│
│  · OAuth start + callback   · plan editor   · status view      │
└───────────────┬────────────────────────────────────────────────┘
                │ HTTP
┌───────────────▼────────────────────────────────────────────────┐
│  API service                                                    │
│  · plans/vibes/slots CRUD   · connection lifecycle              │
└───────────────┬────────────────────────────────────────────────┘
                │
┌───────────────▼────────────────────────────────────────────────┐
│  Postgres                                                       │
│  connections (refresh_token ENCRYPTED, authorized_at, expires_at)│
│  day_plans · slots · vibes · transition_log                     │
│  scheduler leader lock  (pg_advisory_lock — exactly one worker) │
└───────────────┬────────────────────────────────────────────────┘
                │
┌───────────────▼────────────────────────────────────────────────┐
│  Scheduler worker   ← ALWAYS ON. The actual product.            │
│                                                                  │
│  Tick loop, per connection:                                     │
│    1. token valid? refresh if < 5 min left                      │
│    2. GET /me/player    (one call answers everything)           │
│    3. classify → DORMANT | ARMED | APPROACH | BOUNDARY          │
│    4. resolve: which slot should be active at this instant?     │
│    5. decide: nothing / wait-for-boundary / switch now          │
│    6. on switch  → PUT /me/player/play {context_uri}            │
│                    (NO device_id — see §6)                      │
│    7. log outcome to transition_log                             │
│                                                                  │
│  Adaptive cadence:  300s dormant → 60s armed                    │
│                     → 5s approach → 1s boundary                 │
│  Global token bucket + Retry-After obedience + per-user jitter  │
└───────────────┬────────────────────────────────────────────────┘
                │
        ┌───────▼────────┐
        │ PlaybackProvider│  ← the seam (§8)
        └───────┬────────┘
                │
        ┌───────▼────────┐
        │ SpotifyProvider │  ← the only implementation at v1
        └────────────────┘
```

**Why the loop is shaped this way.** One `GET /me/player` answers *both* "did they press play" (P4/TR-3) and "is the track about to end" (P2/TR-1). Those look like two jobs and would cost twice as much if built as two. Keeping them one call is what keeps polling at roughly 1,150 calls per user per day instead of double that — three orders of magnitude inside Spotify's rate limits at five users.

**Why a leader lock.** Two workers polling the same connection would double the API spend and, far worse, could both fire a transition — issuing `PUT /me/player/play` twice, restarting the vibe from track one. A Postgres advisory lock is sufficient and needs no new infrastructure.

**Correctness invariants for the worker** (each one exists because its absence is a bug with an audible symptom):
- **Idempotent transitions.** A slot fires at most once per plan-day per connection; the transition is recorded before it is retried.
- **No fighting the user.** If the observed context already matches the target vibe, do nothing (TR-6).
- **Failure is loud.** `invalid_grant` → stop polling, mark connection broken, tell the user (P6). No silent retry loops.
- **Clock skew is the API's, not ours.** Use `progress_ms` and the response's `timestamp` for boundary maths, never local wall clock alone.

---

## 8. The provider seam

The seam exists to keep Apple Music and Tidal viable later. Its most important job is to be **honest about what they cannot do**, because both are structurally weaker than Spotify here.

```ts
interface PlaybackProvider {
  capabilities(): ProviderCapabilities

  // Reading
  getPlaybackState(conn): Promise<PlaybackState | null>   // null = nothing active
  listVibeSources(conn): Promise<VibeSource[]>            // user-accessible playlists

  // Writing
  startVibe(conn, vibe: Vibe): Promise<void>

  // Credentials
  refresh(conn): Promise<Credentials>
}

interface ProviderCapabilities {
  remoteControl: boolean        // can a SERVER redirect playback? Spotify: true.
                                //   Apple Music: false. Tidal: false.
  serverInitiatedPlayback: boolean
  pushPlaybackEvents: boolean   // webhooks? Nobody: false.
  requiresPremiumTier: boolean
  editorialPlaylistsUsable: boolean  // Spotify: false (C5)
}

interface PlaybackState {
  isPlaying: boolean
  trackId: string
  positionMs: number
  durationMs: number
  contextUri: string | null
  observedAt: number            // for skew-corrected boundary prediction
}
```

**The load-bearing field is `remoteControl`.** When it is false, the scheduler cannot execute transitions at all — the *plan* is still server-side data, but the *execution* has to happen inside a client app that is running and holds the audio session. That is a different runtime topology, not a different adapter, and the seam must make that visible rather than let a future implementer discover it halfway through an Apple Music port.

**What is deliberately kept out of the interface:** anything about devices. No `listDevices`, no `transferTo`. That is not an oversight — device-pinning is the thing P5 forbids and the thing Apple and Tidal could not implement anyway. Keeping devices out of the seam keeps the principle enforceable in the type system.

**What this means concretely for each provider:**

| | Spotify | Apple Music | Tidal |
|---|---|---|---|
| Server can redirect playback | **Yes** (Connect) | **No** — no playback surface in the web API at all | **No** — playback restricted to the Player SDK |
| Where transitions execute | Server | Must be a client app that *is* the player | Same |
| Push events | No — poll | No | No |
| Auto-generated vibes possible | **No** (Recommendations/Audio Features removed) | Technically yes (`recommendations` API exists) | — |
| Realistic next shape | Web + server | **iOS app that plays the music itself** | Deferred |

The Apple Music path is genuinely viable, just differently: an iOS app that is the player can hold a background audio session and schedule its own transitions reliably. It plays out of the phone, so Bluetooth and headphones still work — but a standalone smart speaker does not. **That means P5 is a Spotify-era promise, not a permanent one.**

---

## 9. Non-goals

Explicitly out of scope, with the reason, so none of these get re-litigated:

1. **Cold start** — starting playback from silence. Not possible without an active device (C3), and excluded by design (TR-5).
2. **Auto-generated or inferred vibes.** Not deprioritised — **impossible** (C5). Should not appear on any roadmap.
3. **Spotify editorial/algorithmic playlists as vibes.** Platform-blocked (C5).
4. **Apple Music and Tidal implementations.** Seam only (§8).
5. **A native mobile app.** Web first. Note: an iOS app becomes *required*, not merely nicer, the day Apple Music is attempted.
6. **Shared multi-tenant accounts.** There is no central service and no shared Spotify app registration. Each instance is deployed by one owner, carries that owner's own Spotify application, and can serve **at most five allowlisted Spotify accounts** — a hard platform cap per registration, not a scaling choice (C1). Each connection owns its own plans; there are no household-shared plans or cross-account administration beyond the owner's allowlist.
7. **Crossfade, volume ramping, gapless transitions.** TR-2 makes gapless unachievable through this API.
8. **Per-track or per-artist scheduling.** The unit is a playlist (P3).
9. **Anything commercial.** Prohibited by Spotify platform terms (C1).

---

## 10. Deployment and setup

The product is **self-hosted**: there is no central service to sign up for. Each household deploys its own instance, registers its own Spotify application, and allowlists its own accounts against that registration's five-user cap (C1). Onboarding is therefore "install this," not "sign in with Spotify" — the sign-in step exists, but only after someone has stood up an instance for it to sign in to.

### 10.1 What the person deploying has to do

1. **Register a Spotify application.** Spotify Developer Dashboard → Create app. Record the client ID and client secret, and set the redirect URI to the instance's OAuth callback URL. The account doing this must have Spotify Premium, because "the app owner must have a Spotify Premium account for apps in development mode to function" (C2).
2. **Allowlist the accounts that will use the instance.** Dashboard → app → Settings → User Management → Add new user, with each person's name and Spotify account email. **At most five accounts, ever, per registration** (C1). Because a non-allowlisted account can complete OAuth and then fail every subsequent call with 403, this step is not optional and the app surfaces its omission explicitly (§5, Connect).
3. **Supply the credentials to the instance.** `SPOTIFY_CLIENT_ID`, `SPOTIFY_CLIENT_SECRET`, the OAuth redirect URI, a database URL, and a token-encryption key, provided as environment variables or the host platform's secret store. The client secret and the encryption key must never be committed to a repository or baked into an image.
4. **Deploy** the container and database (§10.2), then open the instance and connect each allowlisted account.
5. **Re-authorize every six months.** The instance will nudge at T‑14 days and hard-warn at T‑3 (C4, P6), but the owner should know at install time that this is a scheduled, recurring task rather than a bug.

Setup documentation ships with the deployment and is part of the deliverable: an install this reader cannot complete is the same as no install at all.

### 10.2 Deployment shape

**What the design requires of any host:** an always-on process (not per-request serverless — the worker holds per-user state and runs 1 Hz bursts); durable encrypted storage for six-month credentials; exactly one scheduler leader; and correct local-time handling across DST.

**Reference deployment: one small always-on container plus Postgres.** This is the shape the instance is built and documented for.

- **Compute:** a single container running two processes, `web` and `worker` — a Fly.io Machine, a Render or Railway service, a small VPS, or a Docker host on the home network. At five users this is on the order of $5–15/month, or free on hardware already running.
- **Data:** Postgres, managed or self-run. Refresh tokens encrypted at rest with AES-GCM under the key from step 3, so a database dump alone is not a credential compromise.
- **Leader election:** `pg_advisory_lock`. No extra infrastructure, and it holds whether one container or two are running.
- **Scheduling:** the in-process tiered loop, **not** system cron. Cron's minute granularity cannot express the 1 Hz boundary window, and shelling out per tick would be absurd.
- **Secrets:** the host's secret store or an untracked env file. `SPOTIFY_CLIENT_SECRET` and the token-encryption key never enter the repository.

**Why not serverless.** A 1 Hz boundary burst as 15 separate function invocations, with per-user state round-tripped through the database each time, is worse in every dimension — cost, latency, and complexity.

**A credible alternative if an instance ever needed to scale:** Cloudflare Durable Objects — one object per connection, alarm-driven wakeups, no idle cost, leader election for free. It is a genuinely better fit for the *shape* of this workload. It is not the reference deployment because a single container is far simpler for someone to stand up and debug, and C1 caps each instance at five users, so scale-out is not the problem self-hosting has.

**Operational must-have: alert on worker liveness.** A dead worker produces **silence**, not errors, and silence is indistinguishable from "nobody is listening right now." This is the failure mode most likely to go unnoticed for days, and the person running the instance is also the person who will notice — so the instance must make it noticeable.

---

## 11. Build sequence

Each step ends somewhere real, and the riskiest thing is proven first.

**0 — Spike (1 day, throwaway). Do this before anything else.**
Register the app, allowlist one account, run Authorization Code by hand, and prove end to end: start playing on a phone → `GET /me/player` reports it → detect the track boundary → `PUT /me/player/play` with a playlist `context_uri` and **no** `device_id` → the phone switches. *This validates C3 and boundary-accurate switching against reality in an afternoon, and every later step depends on both.* If this spike fails, stop and re-plan.

**1 — Auth and connection lifecycle.** Authorization Code, not PKCE (C4); encrypted refresh-token storage; hourly refresh; `expires_at` tracked; `invalid_grant` → broken state. Premium and allowlist checks at connect. *Boring, and the thing that silently kills the product at month six if skipped.*

**2 — Plans, vibes, slots CRUD + playlist picker.** Data model from §4. Picker shows only usable playlists.

**3 — Scheduler worker: the ARMED path.** Leader lock, tiered polling, slot resolution, TR-1/TR-2 boundary detection and switch. **This is the product.** Everything before it was setup.

**4 — Deferred transitions.** TR-3 and TR-4.

**5 — Status, observability, honesty.** `transition_log`, the status view, re-link nudges at T‑14/T‑3, broken-connection states. P6 is a feature and it ships here.

**6 — Deployment story.** Container image, configuration surface, and the setup documentation from §10.1, exercised by a clean install from scratch.

**7 — Harden.** Token bucket, jitter, 429/`Retry-After` handling, playlist-deleted and device-vanished paths, DST edge cases, worker-liveness alerting.

**8 — Extract the seam.** Refactor the Spotify calls behind `PlaybackProvider` *after* the real one works. Extracting an interface from one working implementation beats designing it from two hypothetical ones.

---

## 12. Sources

All fetched and text-extracted 2026-09-24.

- Endpoint restrictions — https://developer.spotify.com/blog/2024-11-27-changes-to-the-web-api
- Quota modes, 5-user cap, extension criteria — https://developer.spotify.com/documentation/web-api/concepts/quota-modes
- Rate limits (rolling 30 s window; limit undisclosed) — https://developer.spotify.com/documentation/web-api/concepts/rate-limits
- Status codes, 429, `QUOTA_EXCEEDED` — https://developer.spotify.com/documentation/web-api/concepts/api-calls
- Premium requirement; `device_id` optional — https://developer.spotify.com/documentation/web-api/reference/start-a-users-playback
- `progress_ms` / `duration_ms` / `is_playing` / `timestamp` — https://developer.spotify.com/documentation/web-api/reference/get-information-about-the-users-current-playback
- Queue takes only track/episode URIs — https://developer.spotify.com/documentation/web-api/reference/add-to-queue
- Transfer requires explicit device — https://developer.spotify.com/documentation/web-api/reference/transfer-a-users-playback
- 6-month refresh token lifetime; `invalid_grant` — https://developer.spotify.com/documentation/web-api/tutorials/refreshing-tokens
- Authorization Code flow — https://developer.spotify.com/documentation/web-api/tutorials/code-flow
- Web Playback SDK Premium + non-commercial terms — https://developer.spotify.com/documentation/web-playback-sdk
- Deprecated banners — https://developer.spotify.com/documentation/web-api/reference/{get-recommendations, get-audio-features, get-audio-analysis, get-an-artists-related-artists, get-featured-playlists}
- Apple Music API topic tree (no player section) — https://developer.apple.com/documentation/applemusicapi
- MusicKit playback locality — https://developer.apple.com/documentation/MusicKit, .../MusicKit/SystemMusicPlayer, .../MusicKit/ApplicationMusicPlayer
- Tidal playback restricted to Player SDK — https://developer.tidal.com/documentation/sdk/api-sdk-overview *(summarised via search index; not fetched directly)*
