# GPS Setter (Zygisk, per-app) — v3, UNTESTED ON DEVICE

Java-only Zygisk module (ZygoteLoader + AndroidVMTools, same stack as HMA-OSS).
Hooks `android.location.Location` getters and `LocationManager.getLastKnownLocation(String)` inside selected apps.
Idea/behaviour taken from Cobits-Code/GpsSetter; Zygisk plumbing taken from HMA-OSS.

## Build
    sh setup.sh                 # fetches AndroidVMTools at the commit HMA-OSS 1fe86ce pins (6fa2d5c)
    ./gradlew :zygote:assembleRelease      # needs JDK 21 + Android SDK 36
    # zip: zygote/build/outputs/magisk/<flavor>/release/GpsSetterZygisk-release*.zip

## Use (root)
    su -c "sh /data/adb/modules/gpssetter_zygisk/gpsz.sh set -6.2088 106.8456 10"
    su -c "sh /data/adb/modules/gpssetter_zygisk/gpsz.sh on"       # off / toggle / random 1 / show
    su -c "sh /data/adb/modules/gpssetter_zygisk/gpsz.sh pkg add com.some.app"   # no rebuild, app is force-stopped
    su -c "sh /data/adb/modules/gpssetter_zygisk/gpsz.sh pkg rm com.some.app | pkg list"
Config lives in Settings.Global `gpsz_cfg` (apps re-read it every ~1 s). Package list = files in
`<module>/packages/` (ZygoteLoader reads them at every app start). A package is injected when its file exists
XOR `.all` exists, so `.all` + `<pkg>` means "all apps except <pkg>" (`pkg all on`; risky).
Runtime-added packages are lost when the module zip is reinstalled — put them in `gpsz.packages` too.

## Changes vs v2
- gradle.properties: android.useAndroidX=true (androidx.annotation dependency fails the build with false; HMA uses true)
- setup.sh pins AndroidVMTools to HMA's commit instead of cloning HEAD
- Java 21 (what AndroidVMTools/HMA use); empty gpsz.packages allowed
- runtime package management (gpsz.sh pkg ...), numeric validation for `set`
- ZygoteEntry: DoNotObfuscate/DoNotShrink annotations like HMA
- config read starts faster (50 ms Application poll, 1 s refresh) -> smaller real-location leak window
- jitter: lat/lng come from the same fix (200 ms snapshot, like Cobits) instead of independent randoms
- stop installing more hooks after the first hard hook failure (HMA's hooksWasCrashed idea)

## Known limits
- Android 13+ only (API 33+); <13 needs HMA's entry-point swapping (not ported)
- Java-level hooks only: no GNSS/native/server-side checks. Callers with already-AOT-inlined getters bypass hooks.
- Real location can leak for the first moments after app start (config not read yet)
- While on, getLastKnownLocation returns a fake fix even without permission / real fix
- Logcat tag `GpsZ` shows injection + hook results

## v0.1.2 (merge dari modul yang sudah jalan di perangkat)
- assets/: gpsz.sh & action.sh versi terbaru (sync/status/check), service.sh, bridge.sh (BARU: v0.1.2), webroot/index.html
- bridge.sh mencari pref app GPS Setter di /data/data, /data/user*, /data/user_de dan /data/misc/*/prefs
  (LSPosed mengalihkan prefs ke /data/misc/apexdata/<uuid>/prefs/...). Tanpa ini status/Start dari app tidak terbaca.
- Matikan modul "GPS Setter" di LSPosed; kalau tidak, app yang sama di-hook dua kali (jalur LSPosed tidak ikut toggle WebUI).
- assets/apk/GpsSetter.apk = APK GPS Setter hasil patch (sama persis dengan yang ada di modul v0.1.1). Dipasang otomatis oleh daemon bridge.sh saat boot kalau app belum ada (`pm install -r`), jadi customize.d/40-install-app.sh tidak lagi wajib; app langsung terpasang setelah reboot pertama.
- APK ini diturunkan dari GpsSetter 1.2.10 (build release LSPosed). Metadata Xposed (xposedmodule, xposeddescription, xposedminversion, xposedsharedprefs) dan assets/xposed_init sudah DIHAPUS, jadi LSPosed tidak lagi mengenalinya sebagai modul. Ditandatangani ulang (APK Signature v2) dengan key baru, jadi versi lama harus di-uninstall dulu sekali.
- tools/build_apk.py + tools/axml_strip.py: membuat ulang APK tanpa Xposed dari APK patched (`python3 tools/build_apk.py <src.apk> <out.apk>`, perlu paket python `cryptography`; key release-key.pem/release-cert.pem dipakai ulang bila ada di folder kerja).

## System-wide (system_server) hook — EXPERIMENTAL, NOT COMPILED OR TESTED
WebUI: Pengaturan -> "System-wide (system_server)" (sama dengan `gpsz.sh system on|off`).
`su -c "sh .../gpsz.sh system on"` then reboot. Creates `packages/android` (ZygoteLoader injects system_server) and sets
Settings.Global `gpsz_sys=1`. In system_server only `SystemHooks` runs (never the app-level Location getter hooks):
LocationManagerService.getLastLocation / injectLocation / GNSS listener blocking, and LocationProviderManager.onReportLocation
(rewrites every reported fix via LocationResult.map). Side effect: system consumers (e.g. time zone detection) also see the fake fix.
Recovery from bootloop: create `/data/adb/modules/gpssetter_zygisk/disable` (adb/recovery), or boot safe mode.
