# GPS Setter Zygisk — gabungan (app tanpa Xposed + modul + hook system-wide)

- `app/`    : GpsSetter (Android1500 v1.3.0-beta) sudah dipatch: tanpa Xposed/YukiHook, tanpa metadata LSPosed.
- `module/` : modul Zygisk (hook per-app + hook system_server + WebUI saklar System-wide).
- `build-all.sh` : build app -> salin APK ke `module/.../assets/apk/GpsSetter.apk` -> build zip modul.

Pakai:  `MAPS_API_KEY=xxxx sh build-all.sh`
Hasil zip modul: `module/zygote/build/outputs/magisk/<flavor>/release/GpsSetterZygisk-release*.zip`

Catatan: belum pernah dicompile/diuji (dibuat tanpa JDK/Gradle). Uninstall GPS Setter lama dulu (tanda tangan berbeda).
System-wide: WebUI -> Pengaturan -> System-wide, atau `gpsz.sh system on`, lalu reboot. Bootloop? buat file
`/data/adb/modules/gpssetter_zygisk/disable` dari recovery/adb.
