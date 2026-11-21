# BitChord source review → Luminous changes

Source reviewed: `I:\download\BitChord-1.9.1.zip` (extracted to `tools/bitchord/`,
not shipped in the APK). BitChord is an AGPL-licensed open-source YouTube Music client —
it is the app behind the reference screenshots (lrc.red credits, frosted glass, animated
canvas/dock).

## What I verified, not guessed
1. `README.md` — playback, downloads, module sources, lyrics providers, Haze glass, crossfade.
2. `data/sources/YouTubeSource.kt` — thin adapter over `StreamResolver`; YouTube is one source.
3. `shared/.../innertube/StreamResolver.kt` (1236 lines) — InnerTubeX + NewPipeExtractor,
   PoToken handling, URL probing, 403/cipher handling.
4. `app/build.gradle.kts` — `NewPipeExtractor v0.26.3` from JitPack, with its conflicting
   `Utils.class` stripped and transitive deps listed explicitly.
5. **Live test:** downloaded NewPipeExtractor 0.26.3 + deps and ran a small Java probe
   (`tools/np-test/TestResolve.java`) against video `J7p4bzqLvCw`. Result: `AUDIO_STREAMS=5`
   with real googlevideo URLs. The URLs contain an `ip=` parameter — proof that resolution
   must happen on the device (a server-resolved URL would 403 from the phone).

## What I ported into Luminous (v6)
- `app/build.gradle.kts`: JitPack repo + stripped NewPipeExtractor jar + its transitive deps.
- `playback/StreamResolver.kt`: on-device video-id → audio URL resolution using
  NewPipeExtractor with an OkHttp-backed downloader.
- `playback/MediaPlaybackService.kt`: `ResolvingDataSource.Factory` resolves
  `luminous://yt/<videoId>` lazily when ExoPlayer loads each queue item — so playback works
  for genuine search results, not just local files.
- `playback/PlayerController.kt` + `domain/model/Music.kt`: remote tracks with a YouTube
  source id are now playable; local/download URIs pass through unchanged.
- `data/repository/LibraryRepository.kt` + `ui/library/LibraryViewModel.kt`:
  `downloadRemote(track)` resolves the stream, writes it to app storage with metadata, and
  adds it to the **Library → Downloads** offline list. Search results now have a ↓ action.
- `ui/navigation/BottomBar.kt`: removed the drag gesture and icon scaling that produced the
  erratic dock motion; a single calm spring now slides the glass pill on tap.

## Honest caveats
- This is an **unofficial** YouTube Music client approach. It may conflict with YouTube's
  Terms of Service and can break when YouTube changes its APIs; BitChord needs PoToken and
  cipher machinery for reliability, which I did **not** port. Stream resolution is therefore
  best-effort and may fail on some videos/networks — failures are surfaced, never faked.
- NewPipeExtractor is AGPL-3.0 (as is BitChord). Luminous now depends on it, so distribution
  obligations may apply. I reused the *approach and dependency*, not BitChord's source files.
- Downloaded provider audio is for personal offline use only. No DRM is bypassed and no
  protected content is decrypted; only publicly served stream URLs are fetched.
- The resolver was validated in a JVM on this machine, **not on an Android device** (none
  available here). On-device runtime success is likely but unverified.

APKs: `deliverables/Luminous-v6-debug.apk`, `Luminous-v6-release-unsigned.apk`.
Both `assembleDebug` and `assembleRelease` succeeded (APK grew ~3 MB with the extractor).
