# LUMINOUS — Final delivery report (fresh v2 build)

Date: 2026-11-20
Project root: `E:\Luminous`
Deliverables: `E:\Luminous\deliverables\Luminous-v2-debug.apk` (22.6 MB),
`E:\Luminous\deliverables\Luminous-v2-release-unsigned.apk` (15.6 MB)

---

## 1. "Exact cause of the original search failure"

Per your instruction, the previous project was scrapped and **not inspected**, so there is no
legacy defect diagnosis to report. This is a clean-room implementation at `E:\Luminous`.

One genuine defect *was* found and fixed inside the new pipeline during construction: the
unifier used a `Track` object as a dictionary key (`if t not in sources`), which is unhashable.
Every `/api/search` call returned HTTP 500. It was fixed to key by
`provider:source_id`, after which all real-query tests passed.

Design decisions that prevent the classic failure modes:
- Both providers are queried with `asyncio.gather`; one provider's failure can never cancel the other.
- Missing Apple credentials produce an explicit `not_configured` status + warning, not an empty list.
- The Android base URL is runtime-configurable (emulator `10.0.2.2`, physical device LAN/HTTPS)
  via an OkHttp interceptor, with cleartext permitted in the network security config.
- HTTP status codes, timeouts, parse errors and provider states are surfaced to the UI with retry.

## 2. Files modified / created (key files)

Backend (`backend/`):
- `app/config.py` — env-only configuration (no secrets in code)
- `app/models/music.py` — provider-agnostic `Track`, `ProviderStatus`, `SearchResponse`
- `app/observability.py` — structured, secret-redacted logging + timers
- `app/services/text.py` — normalization, relevance scoring, cover/tribute rejection
- `app/services/youtube.py` — ytmusicapi 1.12.4 adapter, metadata normalization, view-count ranking
- `app/services/apple.py` — vendored Apple wrapper adapter, explicit not-configured reporting
- `app/services/unify.py` — parallel search, dedupe, source preservation, merge, ranking
- `app/services/cache.py` — TTL cache
- `app/api/routes.py` — `/api/health`, `/api/search`, `/api/explore`, `/api/diagnostics`
- `app/main.py` — CORS, request-ID/status/duration logging, safe error responses
- `vendor/applemusic/` — Myp3a wrapper (vendored; its git-only Widevine dep is not needed)
- `tests/smoke_test.py` — real end-to-end checks
- `run.bat`, `.env.example`, `requirements.txt`

Android (`android/app/src/main/`):
- `java/com/luminous/app/` — `LuminousApp`, `AppContainer`, `MainActivity`
- `data/remote/` — Retrofit client, host-rewriting interceptor, DTOs, mappers, error explanations
- `data/local/` — Room entities/DAOs/database (library, playlists, search history, recents)
- `data/settings/` — DataStore settings
- `data/repository/` — music + library repositories (SAF import, metadata + artwork extraction)
- `playback/` — Media3 `MediaSessionService` and controller
- `ui/theme/`, `ui/components/`, `ui/navigation/` — design system, glass components, floating nav
- `ui/search/`, `ui/home/`, `ui/explore/`, `ui/library/`, `ui/player/`, `ui/settings/` — screens
- `res/` — manifest, cleartext network security config, launcher icons, placeholder artwork

## 3. Backend test results (real provider requests)

Command: `python backend/tests/smoke_test.py` against `uvicorn` (live network).
Final result: **FAILURES: 0** (exit 0).

| Query | HTTP | Results | Top title / artist | Latency |
|---|---|---|---|---|
| blinding lights | 200 | 10 | Blinding Lights / The Weeknd | 2071 ms (cold) |
| joji | 200 | 10 | SLOW DANCING IN THE DARK / Joji | 512 ms |
| doja cat | 200 | 9 | Best Friend (feat. Doja Cat) / Saweetie, then Doja Cat tracks | 656 ms |
| Billie Eilish Birds of a Feather | 200 | 9 | BIRDS OF A FEATHER / Billie Eilish | 695 ms |
| Taylor Swift Cruel Summer | 200 | 10 | Cruel Summer / Taylor Swift | 619 ms |
| Kendrick Lamar HUMBLE | 200 | 9 | HUMBLE. / Kendrick Lamar | 825 ms |

`/api/health`: 200, `status=ok`, version 2.0.0.
`/api/diagnostics?probe=true`: 200, live YouTube probe returned 1 real result in ~567 ms.

Ranking was verified to put the genuine recording first (covers, karaoke, "originally
performed by" variants are penalized). Every returned item carries provider attribution and
preserved source IDs. Partial failure was verified directly: with Apple unconfigured, all
queries still returned YouTube results plus an explicit warning string.

## 4. Android search test results

- `gradlew assembleDebug` → **BUILD SUCCESSFUL** (37 tasks).
- `gradlew assembleRelease` → **BUILD SUCCESSFUL** (49 tasks).
- `aapt dump badging` confirms package `com.luminous.app` v2.0.0, minSdk 24, targetSdk 35,
  and all required permissions (INTERNET, media read, foreground service, notifications).
- Search implementation: 350 ms debounce, in-flight job cancellation, loading spinner,
  provider-specific warning banners, connection-error state with retry, local search history,
  source badges on each result, clickable rows that feed the Media3 queue.

**Not run:** the app itself was not launched or tapped on a device — see item 7.

## 5. Apple Music integration status

- Adapter implemented against `Myp3a/apple-music-api` (vendored; installed from the master
  tarball because `git` is not available on this machine).
- Status: **unavailable in this environment — no `APPLE_MUSIC_DEV_TOKEN` supplied.**
  `/api/health` reports `apple_music = not_configured`, and `/api/search` returns warning:
  *"Apple Music is not configured: set APPLE_MUSIC_DEV_TOKEN in backend/.env."*
- Settings required to enable: a valid Apple Developer token in `backend/.env` and a storefront
  (`APPLE_STOREFRONT`, default `us`). No user token or Widevine device is used, so library
  and DRM download features are intentionally not implemented.

## 6. YouTube Music integration status

- **Working.** ytmusicapi 1.12.4 installed and queried live over the network.
- Real results with genuine `videoId`, `artists`, `album`, `duration_seconds`, `views` and
  thumbnail URLs were verified for all required queries (see item 3).
- Search uses `filter="songs"`, rejects non-song result types and items without a video ID,
  scores relevance against the query, drops off-topic/cover/tribute noise, and ranks by
  relevance then popularity. Largest thumbnail geometry is selected from the real artwork URL.

## 7. Physical-device connectivity status

- **Not verified on hardware.** `adb devices` shows no connected device; there is no AVD,
  no emulator binary, no Android Studio, and virtualization firmware is disabled on this host,
  so an emulator could not be run either.
- The app default base URL is `http://10.0.2.2:8000` (emulator). For a physical device the
  user must enter the PC's LAN address in Settings → Backend connection; the backend must run
  with `--host 0.0.0.0` (provided `run.bat`) and the firewall must allow port 8000.
  Cleartext traffic is permitted for local/LAN development; use HTTPS in production.

## 8. Remaining limitations

- No runtime/UI verification on an Android device or emulator (build + static APK checks only).
- Apple Music is metadata-only and currently disabled due to missing developer token.
- No licensed provider audio streaming: remote results are metadata (+ Apple preview clips when
  configured). Real playback is for user-imported local audio via Media3 only.
- No DRM circumvention, no Widevine, no fabricated lossless labels.
- Explore "mood" chips launch real provider searches rather than curated fake playlists.
- Lyrics show an honest unavailable state; synchronized lyrics are not implemented.
- Release APK is unsigned (no signing keystore was provided).
- Equalizer/audio-output controls are not shown as fake toggles; only standard Media3
  transport, shuffle, repeat, notification/foreground playback are implemented.
