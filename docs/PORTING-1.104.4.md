# Porting 1.98.8 → 1.104.4 / 1.104.5

Status: **built with GitHub Actions and tested on a Fire TV Stick 2nd gen (Fire OS 5.2.9.5)** at
1.104.5 (2026-11-04). The patch series is identical for 1.104.4 and 1.104.5, so it applies to both.

Confirmed on the device (reported by the device owner):
- installs, launches, logs in, tunnel comes up, stays logged in across reboots;
- **boot auto-start (patch 0009) works** — after a reboot, with no launch-on-boot helper app
  installed and nothing opened, Tailscale and its `tun` interface come up on their own;
- **running the stick as an exit node works** — another device's traffic relayed through it
  (`wlan0` rx/tx rose by ~42 MB / ~44 MB together); throughput is low on 2.4 GHz Wi-Fi, as expected.

Checked by script: all patches apply in order to pristine upstream. Items under
[Not yet checked](#not-yet-checked) remain open.

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
| 0009 | **new, optional** — start the VPN after boot (`BootReceiver`); there is no always-on VPN below API 24. Only starts if logged in and VPN permission already granted. Delete this patch to disable | 24 |

*Verified:* all nine apply in order to pristine `1.104.4` and reproduce the intended tree
byte-for-byte; `build.sh` passes `bash -n`; its minSdk override was exercised against both the
new `.kts` and the old Groovy file.

## Not yet checked

Resolved by the device test above: the full build and the Kotlin compile, launch / render on
GLES 2.0, login, tunnel, and boot auto-start (0009).

Still open:

1. **Using an exit node from the stick.** STATUS.md verified this on 1.98.8. Not re-measured on
   1.104.x (note the tunnel interface may be `tun1`, not `tun0`, if another VPN app held `tun0`).
2. **Expedited work.** Upstream now provides `getForegroundInfo()`, so WorkManager's expedited
   path might work on API 22, but 0006 deliberately keeps the already-verified ordinary-work
   path.
3. **Notifications.** New key-expiry and Taildrop notifications check
   `POST_NOTIFICATIONS`, which API 22 does not define; they may be silently skipped. This is
   the same pattern the health notifier already used, not a crash.
4. **Boot-start timing.** Auto-start works, but the console took a few minutes to show the node
   connected after a reboot. Not measured; may just be the stick's slow boot.
5. **Known upstream issue, not touched:** `StartVPNWorker`'s "not ready" path builds a
   `Notification.Builder(context, channelId)` (API 26) and would crash on Fire OS 5 if reached.
6. **Reinstalling.** A build signed with a different key needs `adb uninstall com.tailscale.ipn`
   first (login is lost). The Build APK workflow signs with a permanent key from repo secrets
   (see STATUS.md "Signing"), so later builds update in place with `adb install -r`.

Tips learned on device: check the tunnel with `adb shell ls /sys/class/net | findstr tun` rather
than assuming `tun0`; when the stick *is* the exit node, relayed traffic shows on `wlan0`
rx/tx, not on the tunnel interface.

## Rolling forward again

`build.sh` looks for `patches/<full ref>/`, then `patches/<version>/`. To move to a newer
release, copy the newest directory, run `git apply --check` on each patch against the new
ref, and re-audit anything upstream added since (the quickest signal is
`diff -ru` of `android/src/main` between the two tags, grepping for API 23+ calls).

## Update to 1.104.5 (2026-11-04)

Upstream `1.104.5-t788384763-gd8673104f` changed only `go.mod` / `go.sum` (`tailscale.com`
v1.104.4 → v1.104.5). Nothing under `android/` changed, so `patches/1.104.5/` is a verbatim copy
of the 1.104.4 series. *Verified:* all nine apply in order to pristine 1.104.5 and produce an
`android/` tree identical to the 1.104.4 tree audited above. Compile and on-device checks listed
above still apply. `patches/1.104.4/` is kept as a fallback.
