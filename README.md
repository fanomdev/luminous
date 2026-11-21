# Luminous — an independent music app

Luminous is an independent Android music application built around one idea: unified discovery across Apple Music and YouTube Music, with an honest, compliant client.

This repository is the **public project page**: documentation and project assets only. The application source and compiled APK are not published here.

## What it does

- **Unified discovery** — a FastAPI service normalizes and merges metadata from Apple Music and YouTube Music into one search result set.
- **Cloudflare gateway** — a TypeScript Worker provides the public HTTPS entry point and keeps provider secrets server-side.
- **Native player** — Kotlin + Jetpack Compose client using Media3, Room, DataStore, Retrofit and Coil, with a liquid-glass UI.
- **Lyrics and playback state** — lyrics lookup and play/pause/seek/queue state synced across the app.
- **Compliance** — metadata-only for remote providers. No DRM circumvention, no stream extraction, no unauthorized downloads. Local playback is for user-owned imported audio.

## Documentation

- `docs/PHASE3_DELIVERY_REPORT.md` — gateway, lyrics and glass phase report
- `docs/FINAL_DELIVERY_REPORT.md` — final delivery notes
- `docs/BITCHORD_REVERSE_ENGINEERING.md` — companion research note

## Status

Independent project. Source and binaries are intentionally not distributed from this repository.
