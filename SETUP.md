# DFRoot — Start at Boot fix

## What was changed

**`app/src/main/java/df/root/BootReceiver.java`** — the only modified file. Two lines added:

1. `final PendingResult async = goAsync();` — tells Android the receiver is still working after `onReceive()` returns, keeping the app process alive.
2. `async.finish();` — called in the `finally` block when the DirtyFrag exploit finishes (success or failure), releasing the system's hold on the process.

## Root cause

`BootReceiver.onReceive()` launched the DirtyFrag exploit on a background thread and returned immediately. Android kills the app process as soon as `onReceive()` returns, so the exploit thread was killed mid-execution — before `kernel_module_exploit()` could run `insmod` on `dfroot.ko`. The partial wake lock kept the CPU awake but did **not** prevent process death. `goAsync()` is the standard Android mechanism for asynchronous work in a broadcast receiver.

## How Start at Boot works

`SettingsFragment` toggles `BootReceiver` via `PackageManager.setComponentEnabledSetting()`. The manifest marks it `android:enabled="false"` by default (opt-in). On boot, `LOCKED_BOOT_COMPLETED` or `BOOT_COMPLETED` fires the enabled receiver, which takes a partial wake lock and runs `ExploitRunner.run()` on a thread. The native DirtyFrag exploit patches `libc++.so`/`libbase.so` so init (uid 0, tid 1) triggers shellcode when reaping an orphan; the shellcode `insmod`s `dfroot.ko`, which sets SELinux permissive and runs `bootstrap.c`. Bootstrap reads the saved SU manager and launches `ksud late-load --package-name <manager>`.

## How Auto Soft Reboot works

The `soft_reboot` pref lives in device-protected `dfroot.xml` and is read by `bootstrap.c:read_prefs()`. After `ksud late-load` succeeds, if enabled, bootstrap waits for the late-load `ksud` to exit, then runs `ksud soft-reboot` in the `u:r:ksu:s0` context to restart into root. Applies to both manual start and start-at-boot.

## How Disable KSU Modules works

The `disable_modules` pref is read by bootstrap. When on, `disable_modules()` walks `/data/adb/modules/` and touches `<module>/disable` for every installed module *before* launching ksud, so a broken module won't load and bootloop the device. Recovery: boot to Safe Mode, enable "Disable KSU Modules", start root, then remove the bad module and turn the setting off.

## Extracting the .ko files from the release APK

The DFRoot release APK (`DFRoot_4.7.apk`) ships `libdfroot.so` with all 8 kernel modules `.incbin`'d into `.rodata`. To extract them:

1. Extract `lib/arm64-v8a/libdfroot.so` from the APK zip.
2. Scan the binary for ELF magic bytes (`0x7f 0x45 0x4c 0x46`) — each `.ko` is a complete ELF file placed back-to-back in `.rodata`.
3. Split the binary at each ELF header boundary; each segment between consecutive headers is one `.ko` file, named `dfroot-android<X>-<Y>.ko` matching the `incbin` labels in `app/src/main/jni/exp.c`.
4. Place the 8 `.ko` files in `app/src/main/jni/ko/` so CMake's dependency check passes during Gradle builds.

Note: this extraction is only needed for local Gradle builds. The release APK already contains the `.ko` files embedded in `libdfroot.so` — you do not need podman/DDK containers unless you want to rebuild the modules for a different kernel version.

## Battery optimization (Honor/MagicUI)

Honor devices aggressively restrict background broadcasts for non-whitelisted apps. Even with `goAsync()` fixed, Start at Boot will not fire until you disable battery optimization for DFRoot: **Settings → Battery → Battery optimization → DFRoot → Don't optimize.**

## Build

Requires Android SDK (platforms/android-37, build-tools/37.0.0) and NDK r27+. The `.ko` files must be present in `app/src/main/jni/ko/`.

```powershell
.\gradlew.bat :app:assembleRelease --no-daemon
```

Output: `app/build/outputs/apk/release/DFRoot_4.7.apk`. Install via `adb install`.
