# LUMINOUS — Phase 3 delivery report

## Follow-up v6 (BitChord source review)
- Reviewed the supplied `BitChord-1.9.1.zip`; confirmed it is the app in the reference shots.
- Verified NewPipeExtractor 0.26.3 resolves real, IP-bound YouTube audio streams in a JVM test.
- Ported on-device lazy stream resolution into Luminous (NewPipeExtractor + Media3
  `ResolvingDataSource`), so genuine search results now play through ExoPlayer.
- Added ↓ download on search results: resolves the stream, saves it to app storage, and shows
  it in Library → Downloads alongside imported local files.
- Simplified the dock to one calm spring pill (removed the erratic drag/scale behaviour).
- Full details and caveats: `docs/BITCHORD_REVERSE_ENGINEERING.md`.
- APKs: `Luminous-v6-debug.apk`, `Luminous-v6-release-unsigned.apk`; both builds succeeded.
- Caveat: unofficial YouTube client approach, AGPL dependency, PoToken machinery not ported,
  and no on-device verification (no device available).

## Follow-up v5 (user-reported breakages)
- **Shine removed:** the moving specular reflection was deleted from `GlassSurface`, the
  reflection preference/slider was removed entirely, and glass is now a calm translucent fill
  with a static subtle edge. No animated sheen anywhere.
- **Settings toggles:** replaced the generic unchecked-cast DataStore write with typed writes,
  and made every setter update the UI state *immediately* before persisting, with a brief
  "✓ Setting saved" chip. Switches no longer appear dead.
- **Player opening animation:** the previous `AnimatedVisibility(visible = true)` sat on a
  composable that was only added when expanded, so it never animated. The visibility animation
  now lives in `MainActivity` around `FullPlayer` with slide + fade + scale enter/exit.
- **Playback fixes:** controller connection now queues a pending play request on cold start;
  queue index maps correctly to the filtered playable list; tapping a metadata-only remote track
  shows an explicit in-player notice instead of silently doing nothing; local/downloaded audio
  plays through Media3.
- **Repeat/shuffle:** real mode cycling with visible state (repeat-off/all/one and shuffle tint).
- **Downloads / offline library:** new Room table + repository copies user-owned audio into app
  storage, persists metadata/artwork, and exposes a **Library → Downloads** tab with Add music,
  play, remove, and offline count. No provider audio is downloaded or ripped.
- **Lyrics UI:** removed the drifting particles and simplified the layout; kept animated
  current-line highlighting, glass attribution and animated source chips.
- Builds: `assembleDebug` and `assembleRelease` both BUILD SUCCESSFUL; APKs `Luminous-v5-*.apk`.
- Not device-tested (no device/emulator). Playback of provider metadata results remains
  unavailable by design; use Downloads for real audio.

## Follow-up (404 + animation pass)
- **Root cause of the "Backend endpoint not found (HTTP 404)" error:** the Android app calls
  `GET /api/explore`, but the Worker did not implement that route. Added `/api/explore` to
  `cloudflare/src/index.ts` backed by real `ytmusic-api` home sections (30 genuine items),
  redeployed the Worker, and verified live: `/api/explore` 200, full test suite 0 failures.
- Animations added to match the supplied reference visuals:
  - Dock: a liquid-glass highlight pill that springs/slides between tabs on tap and while
    dragging horizontally; icons scale and animate color on selection.
  - Mini-player: pops in with spring slide + scale, and crossfades/scales when the track changes.
  - Full player: blurred artwork crossfades on track change; play/pause icon scales+fades;
    controls slide/fade; artwork pops in.
  - Lyrics: animated slide transition into the lyrics view, animated source chips, drifting
    decorative particles, current-line highlight with animated size/offset/accent bar.
- Rebuilt APKs: `deliverables/Luminous-v4-debug.apk` and `Luminous-v4-release-unsigned.apk`.
  Both `assembleDebug` and `assembleRelease` returned BUILD SUCCESSFUL.
- Still not device-tested (no device/emulator); the animations were compiled and reviewed, not
  visually verified on hardware.


Scope: repair backend/API connectivity, add Cloudflare Workers gateway, real Liquid Glass,
functional settings/themes, multi-source synchronized lyrics, player integration, diagnostics.
The existing v2 design was preserved — no redesign or new project.

## 1. Discovered problems (from inspection, not guessed)

1. **Backend was emulator-only.** Default base URL `http://10.0.2.2:8000` + a backend started on
   `127.0.0.1` meant physical devices and any public client could not reach the API.
2. **No Cloudflare deployment existed**, so there was no HTTPS public endpoint and no place to
   hold provider credentials securely.
3. **Accent selection was inert.** `LuminousTheme` applied dynamic system colors whenever
   `dynamicColor=true` (the default), overwriting the chosen accent.
4. **Only a handful of settings existed**; most appearance/sound/download/lyrics controls were
   absent, and the few present were not wired into rendering.
5. **Lyrics were a hardcoded placeholder** in the full-screen player; no provider, model,
   cache, endpoint, or playback synchronization existed.
6. **Glass surfaces were flat translucent gray cards** with no specular border, moving
   reflection, depth or real blur.
7. **Python backend lacked** `/api/lyrics`, `/api/providers/status`, `/api/artist/:id`,
   `/api/album/:id`, and did not accept the `APPLE_DEV_TOKEN` env name.
8. **No provider credentials exist** in any env file or environment variable on this machine —
   only the Cloudflare API token supplied in the request (used for deployment, never logged
   or stored in files).

## 2. Fixes by area

### Cloudflare / connectivity
- New TypeScript Worker at `E:\Luminous\cloudflare` implementing the full normalized API.
  It runs natively on Workers (FastAPI cannot), keeping the Python adapters as the local engine.
- Deployed and verified: **https://luminous-gateway.ayanbasit269.workers.dev**
- Android default base URL changed to that HTTPS endpoint; Settings can override for local/LAN.
- All provider secrets are Worker bindings. No credentials were uploaded because none exist;
  missing providers report `not_configured` honestly.

### Python backend (subagent-verified)
- `config.py` accepts both `APPLE_MUSIC_DEV_TOKEN` and `APPLE_DEV_TOKEN`; added Musixmatch /
  Genius keys and lyrics TTL; secrets marked `repr=False`.
- New `services/lyrics.py`: LRCLIB, Musixmatch, Genius adapters; variant-aware matching;
  synced > plain > link ranking; TTL cache; independent provider failures.
- New routes: `/api/lyrics`, `/api/providers/status`, `/api/artist/{id}`, `/api/album/{id}`;
  `/api/health` now lists 5 providers.
- Live test (`backend/tests/lyrics_test.py`): **0 failures**, 35 synced lines returned, cache hit
  on repeat, missing credentials reported as `not_configured`.

### Liquid Glass
- `ui/glass/Glass.kt`: layered translucent fill, animated specular band, gradient reflective
  border, depth shadow, and real backdrop blur via Compose (RenderEffect where supported).
- Applied through `AppGlass` (settings-driven) to bottom navigation, mini-player, settings
  panels, lyrics source chips and dialogs; all existing glass got gradient borders/highlights.
- Full-screen player now blurs the actual artwork behind the controls and derives color with Palette.

### Settings (all persisted in DataStore)
- Themes: system / dark / light / **AMOLED**, plus dynamic-color and custom ARGB accents;
  accent now overrides system colors.
- Glass: intensity, transparency, blur, border brightness, reflection intensity.
- Animation: intensity, reduced motion, smooth transitions, artwork animation, glass animation.
- Layout: compact layout, rounded artwork, interface density.
- Sound: gapless (Media3 default with `setMediaItems`), working volume crossfade
  (0–12 s), loudness normalization, equalizer preset, bass boost + strength, audio-focus behavior.
  Real Android `Equalizer`/`BassBoost`/`LoudnessEnhancer` applied in the Media3 service when the
  device supports them; failures are logged, never silently faked.
- Downloads/library: storage location, offline-only preference, authorized-quality preference,
  real storage usage, imported-track count, and remove-imported-references action.
- Lyrics: enable/disable, preferred source, auto-scroll toggle.
- Diagnostics panel: real test request, provider status, last success/error, response time.

### Lyrics + player
- Room `lyrics_cache`, `LyricsRepository` (Moshi + TTL), `LyricsViewModel` (track-change reload,
  cancellation, provider preference, manual source selection).
- Immersive lyrics screen with artwork-derived background, real blur, synced-line highlighting,
  auto-scroll to current line, return-to-current action, plain-lyrics mode, source attribution,
  retry, and honest unavailable state. Uses **actual Media3 position**, never a fake timer.
- Lyrics reload on track change so the previous song's lyrics never persist.

## 3. Test results

| Check | Result |
|---|---|
| Worker local tests (`tests/worker_test.py`) | 0 failures; 5 real queries, synced lyrics, probe |
| Worker deployed tests (live HTTPS) | 0 failures; same queries, 35 synced lines |
| Python backend lyrics/status tests | 0 failures |
| `gradlew assembleDebug` | BUILD SUCCESSFUL |
| `gradlew assembleRelease` | BUILD SUCCESSFUL (after freeing memory from dev daemons) |
| APK artifacts | `deliverables/Luminous-v3-debug.apk`, `Luminous-v3-release-unsigned.apk` |

Example deployed results: "Blinding Lights", "Glimpse of Us", "BIRDS OF A FEATHER",
"Agora Hills", "Cruel Summer" all returned genuine YouTube Music metadata with correct
artists; Apple reported `not_configured` and YouTube results were unaffected.

## 4. Provider status

- **YouTube Music: working** (ytmusic-api on Worker; ytmusicapi 1.12.4 in Python).
- **LRCLIB: working** — real synced lyrics with attribution, no key required.
- **Apple Music: not configured** — no developer token available; reported explicitly.
- **Musixmatch / Genius: not configured** — no API keys available; adapters exist and are
  documented but their success paths cannot be tested here.

## 5. Limitations / honest notes

- **No device testing.** No connected adb device, no AVD, no emulator binary, and hardware
  virtualization is disabled, so the APK was compiled and statically verified but never launched.
- Provider-key setup remains a single external action: add the needed token to your Cloudflare
  account, then `npx wrangler secret put <NAME>` and redeploy. No key was invented.
- Glass uses Compose blur (native RenderEffect on modern Android) rather than a custom GPU
  shader; reflections are drawn, not physically refracted.
- Equalizer presets map to the device's available presets; unsupported effects are logged and
  skipped. Audio-quality labels remain preference-only for remote tracks.
- Release APK is unsigned (no keystore supplied).
- Settings persistence was verified by code path and DataStore wiring, not by restarting a
  running app on a device.
