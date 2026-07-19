# Flowly Kernel (libbridge.so)

This repository **builds and publishes Flowly's kernel** — the `libbridge.so` native
library — for the [Flowly](https://github.com/SkyAlice-source/flowly-android) Android app.

## Why this exists

Flowly loads its proxy core (`mihomo`) through a JNI library, `libbridge.so`, which is
normally **baked into the APK at build time**. That means the kernel version is frozen
until the next app release.

This repo lets the app **download a newer `libbridge.so` at runtime and hot-swap it on
restart**, so users can try newer (or different) mihomo versions without waiting for an
app update.

## How it works

`libbridge.so` is produced by Flowly's own `core` module (the `golang-android` Gradle
plugin). The JNI interface (`cfa/native`) is part of Flowly's source, so a rebuild is
**100% ABI- and interface-compatible** with the installed app — no `UnsatisfiedLinkError`,
no rewrite of the core layer.

For each channel we simply swap the **vendored mihomo source**
(`core/src/foss/golang/clash`) to a different mihomo version and rebuild:

| Channel   | mihomo source                                   |
|-----------|-------------------------------------------------|
| `stable`  | latest stable `MetaCubeX/mihomo` release tag    |
| `latest`  | latest stable `MetaCubeX/mihomo` release tag    |
| `alpha`   | `MetaCubeX/mihomo` `master` branch              |

(Override any channel's ref via the workflow `workflow_dispatch` inputs.)

## Build & publish (CI)

`.github/workflows/build.yml` runs on `workflow_dispatch` (and on push to `kernel-build`):

1. Check out `SkyAlice-source/flowly-android`.
2. Resolve the mihomo ref for the channel.
3. Replace `core/src/foss/golang/clash` with that mihomo version.
4. Build `./gradlew :app:assembleMetaDebug` (requires Android SDK + NDK 28.2.13676358 + Go 1.22 + CMake).
5. Extract `libbridge.so` for all 4 ABIs.
6. Compute `sha256.txt`.
7. Publish a GitHub Release tagged `stable` / `latest` / `alpha` containing:
   - `libbridge-arm64-v8a.so`
   - `libbridge-armeabi-v7a.so`
   - `libbridge-x86_64.so`
   - `sha256.txt`

## Asset / ABI contract (consumed by the app)

- The app detects its ABI via `Build.SUPPORTED_ABIS` (preferring `arm64-v8a`).
- It downloads `libbridge-<abi>.so` from the matching channel release.
- It verifies the file against the `sha256.txt` manifest before installing.
- On install it writes the `.so` to `filesDir/kernel/libbridge.so` and **restarts the app**;
  `Bridge.loadNativeLibrary()` then prefers that file over the bundled one.

## Security

The `.so` is fetched from **this repository's own Releases** (not a third party). The app
verifies a SHA-256 checksum from the accompanying `sha256.txt` before loading, so a
corrupted or tampered artifact is rejected and the app falls back to its bundled kernel.

## Toolchain versions (must stay aligned with flowly-android)

- Android NDK `28.2.13676358`
- Go `1.22` (module declares `go 1.20`)
- `golang-android` Gradle plugin `1.0.4`
- Android platform `35`, build-tools `35.0.0`, CMake `3.22.1`
