# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A casual mobile snake game — **꼬마 뱀: 사과 모험** ("Little Snake: Apple Adventure") — packaged as an Android app via [Capacitor](https://capacitorjs.com/). The entire game is a single self-contained HTML file (`www/index.html`); Capacitor wraps it in a native Android WebView shell for distribution on Google Play.

App ID: `com.leehojun.cutesnake` (configured in `capacitor.config.json` and `android/app/build.gradle`).

## Architecture

The codebase has two distinct layers:

**1. The game (`www/index.html`)** — This is where ~all logic lives. A single ~1900-line file containing HTML, CSS, and vanilla JavaScript with **no build step, no framework, and no dependencies**. Key pieces:
- **Rendering**: HTML5 `<canvas>` 2D context, DPR-aware, sized to a fixed 450×800 play field. The snake is a free-moving path (`path[]`, newest-first) sampled at fixed spacing rather than a grid — collision uses head/body radii (`HEAD_R`, `BODY_R`).
- **Game loop**: `loop(t)` → `update(dt)` + draw, driven by `requestAnimationFrame`. Bootstrapped at the bottom of the file.
- **Round system**: progressive difficulty via `round*` helper functions (`roundGoalLength`, `roundStartTime`, `roundRockCount`, `roundBeeCount`, ...) that scale obstacles/speed/time by round number. Between rounds there's a "door" transition (`updateDoorPhase`) and per-round board themes.
- **Audio**: synthesized at runtime with the Web Audio API — `tone()`/`noise()` for SFX (`sfxEat`, `sfxCrash`, etc.) and a sequenced chiptune BGM (`BGM_MELODY`/`BGM_BASS`, `bgmTick`). No audio asset files.
- **Persistence**: `localStorage` only, all keys prefixed `snake_` (`snake_bestscore`, `snake_skin`, `snake_leaderboard`, `snake_streak`, `snake_lastclaim`, `snake_tutorial_done`, `snake_mute`). Every read/write is wrapped in `try/catch`. The leaderboard is local-only (no backend).
- **Haptics**: uses the raw `navigator.vibrate()` Web API directly — **not** a Capacitor plugin.
- **Input**: touch swipe is primary; arrow keys work on desktop; an on-screen d-pad (`#dirPad`) exists but is hidden by default.

**2. The native Android shell (`android/`)** — Standard Capacitor-generated Gradle project. `MainActivity.java` is an empty `BridgeActivity`; you should rarely touch native code. `npx cap sync` copies `www/` into `android/app/src/main/assets/public` and regenerates plugin wiring — so **native asset changes are generated, not hand-edited**.

> The practical consequence: nearly all game changes happen in `www/index.html`. The `android/` tree is build infrastructure.

## Commands

There is no web build and no test suite (`package.json`'s `test` script is a placeholder that fails). Workflow:

```bash
npm install                  # install Capacitor CLI + platform packages
npx cap sync android         # copy www/ → android assets, sync plugins (run after ANY www/ change)
npx cap open android         # open the project in Android Studio
```

Build a debug APK from the command line:

```bash
npx cap sync android
cd android && ./gradlew assembleDebug --no-daemon
# output: android/app/build/outputs/apk/debug/app-debug.apk
```

To preview the game logic alone, open `www/index.html` directly in a browser — it runs standalone (touch/keyboard input, localStorage, and audio all work without Capacitor).

## CI / releases

`.github/workflows/android-build.yml` builds a **debug** APK on every push to `main` (and via manual `workflow_dispatch`), uploading it as the `cutesnake-debug-apk` artifact. Toolchain: Node 22, JDK 21 (temurin), Android SDK. The release `.aab` for Google Play is **not** produced by CI — it's built manually in Android Studio with a signing keystore (see `빌드_실행_가이드.md`).

`android/variables.gradle` pins SDK levels: `minSdk 24`, `compile/targetSdk 36`.

## Conventions

- **In-app text, code comments, and the two `.md` guides are written in Korean.** Match the existing language when editing game-facing strings or comments.
- When you change `www/index.html`, the change won't reach the app until `npx cap sync` runs. Don't edit files under `android/app/src/main/assets/` directly — they're overwritten by sync.
- `versionCode`/`versionName` for releases live in `android/app/build.gradle`.
- The two Korean docs (`빌드_실행_가이드.md` build guide, `출시_진행상황.md` release-status checklist) reference a Windows path (`C:\Users\151934\...`) from the original author's machine — treat those paths as illustrative, not literal.
