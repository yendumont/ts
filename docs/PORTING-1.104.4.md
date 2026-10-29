# Porting 1.98.8 → 1.104.4

Status: **patches ported and verified to apply; APK not yet built or tested on a device.**
The build needs the Android SDK/NDK and Maven/Gradle downloads, which were unavailable in
the environment where this port was prepared. Everything below that is marked *verified*
was checked by script; everything under [Still unverified](#still-unverified) needs a real
build and a Fire TV Stick.

Target: `1.104.4-t938213198-gb9d487fe6` (commit `b9d487fe6d72`, 2026-10-27). Stable releases
use even minor numbers, so 1.104.x is the current stable line; 1.105.x is unstable.

## What changed upstream that mattered

| Change | Impact | Handled by |
|---|---|---|
| Groovy `build.gradle` → Kotlin DSL `build.gradle.kts` (`minSdk = 26`) | the old `minSdkVersion` sed and patch 0007 no longer matched | `build.sh` handles both layouts; 0007 ported |
| AGP 8.6.1 → 9.4.0, Gradle 8.7 → 9.6.0, Kotlin 1.9.22 → 2.2.10 | toolchain only; runtime libraries are **identical** to 1.98.8 (same androidx, serialization 1.6.3, coroutines 1.8.1, security-crypto alpha06, WorkManager 2.9.1) | nothing needed |
| compileSdk 34 → 36, build-tools 36.0.0 | `build.sh` hardcoded 34 | `build.sh` now reads `androidApiLevel` / `androidBuildToolsVersion` from the checkout |
| Go 1.26.3 → 1.27.3 | Go 1.27 release notes mention no Android API change | none (still verify with a build) |
| `setExpedited` now at **4** call sites (was 3) | an unpatched site would crash on API 22 | 0006 extended to the new debug-only integration-login site |
| `VpnService.Builder.setUnderlyingNetworks(emptyArray())` when no network is cached | on API 22 the network callback never fires, so the cache is always empty; empty means "no underlying network" rather than "system default" | new **0008** passes `null` below API 23, as 1.98.8 did |
| `decodeFromStream` also used in `ui/localapi/Client.kt` | same `CharsetDecoderICU` bug as 0005 for responses > 16 KB (latent in 1.98.8 too) | 0005 extended to `Client.kt` |
| New: `WorkerForegroundInfo`, key-expiry notifications, Taildrop inline share, deep links | audited: only `java.time` (desugared) and API-guarded notification/permission checks | nothing needed |

## Patch series (`patches/1.104.4/`)

| # | Fixes | API |
|---|---|---|
| 0001 | `NotificationChannel` guard (covers all four channel creations) | 26 |
| 0002 | `QuickToggleService : TileService` guards | 24 |
| 0003 | `startForegroundService` / `PendingIntent.getForegroundService` | 26 |
| 0004 | `Network.bindSocket` → `VpnService.protect(fd)` fallback | 23 |
| 0005 | JSON decode from string, not stream (`Notifier.kt` **and** `Client.kt`) | ≤ 23 |
| 0006 | drop `setExpedited` (4 sites) | 31 |
| 0007 | `coreLibraryDesugaring` for `java.time` (Kotlin DSL) | 26 |
| 0008 | **new** — underlying networks `null`, not empty, below API 23 | 23 |

*Verified:* all eight apply in order to pristine `1.104.4` and reproduce the intended tree
byte-for-byte; `build.sh` passes `bash -n`; its minSdk override was exercised against both the
new `.kts` and the old Groovy file.

## Still unverified

1. **A full build.** Run `./scripts/build.sh`, or use the *Build APK* workflow in Actions.
   `build.sh` hard-fails if `verify-apk.py` finds minSdk above 22 or a missing ABI.
2. **Kotlin compile of the patched sources** under Kotlin 2.2.10 / AGP 9.4.
   `desugar_jdk_libs:2.1.5` in 0007 was chosen for AGP 9 compatibility but not resolved here.
3. **On-device behaviour on Fire OS 5** — specifically: launch and render on GLES 2.0, login,
   tunnel up, peer list populated, exit-node routing (`tun0` byte counters, see STATUS.md),
   and no `NoSuchMethodError` / `NoClassDefFoundError` / `VerifyError` in logcat.
4. **Expedited work.** Upstream now provides `getForegroundInfo()`, so WorkManager's expedited
   path might work on API 22, but 0006 deliberately keeps the already-verified ordinary-work
   path.
5. **Notifications.** New key-expiry and Taildrop notifications check
   `POST_NOTIFICATIONS`, which API 22 does not define; they may be silently skipped. This is
   the same pattern the health notifier already used, not a crash.
6. **Reinstalling.** A new APK signed with a different key needs `adb uninstall
   com.tailscale.ipn` first (login is lost); sign with the durable key (STATUS.md) to avoid
   this on future upgrades.

## Rolling forward again

`build.sh` looks for `patches/<full ref>/`, then `patches/<version>/`. To move to a newer
release, copy the newest directory, run `git apply --check` on each patch against the new
ref, and re-audit anything upstream added since (the quickest signal is
`diff -ru` of `android/src/main` between the two tags, grepping for API 23+ calls).
