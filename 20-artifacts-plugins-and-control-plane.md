# 20 — Artifacts, Plugins & the Guest Control Plane

Completes the systematic coverage of every file in `subjects/`: the decrypted
plugin overlays, the VMOS Pro assets, the guest-side control service
(`VMServer`), the framework-plugin loader, and a full coverage tracker.

Sources: `workspace/plugins/` (decrypted overlays), `workspace/vmospro_assets/`,
`workspace/superuser_apk/`, `workspace/jadx/vmserver/`,
`workspace/jadx/vmfwplugin/`, `workspace/notes/vmaster_guest_deepdive.md`.

---

## 1. Subjects coverage tracker

| Subject file | Analyzed in |
|---|---|
| `subjects/twoyi/` (source + `.git`) | docs 03, 09, 15; `twoyi_guest_deepdive.md`; git log (README/translations only) |
| `subjects/twoyi_0.5.4-*.apk` | docs 03, 09, 10, 13; `workspace/apk/twoyi`, `workspace/twoyi_rootfs` |
| `subjects/VM-v3.2.80-official.apk` (Virtual Master) | docs 04, 19; `virtualmaster_java_deobfuscated.md`, `vmaster_guest_deepdive.md`; this doc (§2) |
| `subjects/vmospro.apk` | docs 05, 09, 11, 12; `vmospro_java_findings.md`, `vmospro_native_deepdive.md`; this doc (§3) |
| `subjects/rom_11_0_0_1036_3280.zip` | docs 12, 19; decrypted to `workspace/vmaster/fs`; this doc (§4) |

Sub-artifacts swept this round:

| Artifact | Finding |
|---|---|
| Virtual Master `assets/plugins/{magisk,superuser,xposed,play}.zip` | **AES-ECB** (`?h=true`, key `%z89aviCM0KkbEs9`) → Magisk, ChainsDD Superuser, Xposed, Google Play/GMS (§2) |
| `rom_11_0_0_1036_3280.zip!/superuser.zip` | AES-ECB → Superuser overlay (§2.1) |
| VMOS Pro `assets/vmfwplugin` | framework-plugin **loader** (`com.android.framework.plugin`) → `/data/plugins/` (§3.2) |
| VMOS Pro `assets/romex-framework-plugin.apk` | a full **Xposed framework** + system-service hooks (location/vibrator/GNSS) + AXML/ARSC (§3.3) |
| VMOS Pro `assets/romex_cn.apk` (`com.android.secex`) | the **guest-side RomEx service** (`com.vmos.exsocket.*`, native `libvm_romex.so`) (§3.4) |
| VMOS Pro `assets/{Launcher3,vmos-pro-intent*,romex_cn}.apk`, `pubkey`, `siminfo.json` | launcher, launcher-intent, ROM extension, 64-byte key/hash, SIM/carrier DB (§3) |
| Virtual Master guest `vmserver-service.jar` | the guest **`VMServer` system service** (control plane over `/dev/vm_event`) (§4) |
| twoyi `libadb.so` | a statically-linked **adb 31.0.3** binary |
| twoyi `libloader.so` | MBA-obfuscated; init arrays use `-1` sentinels (non-standard) → internals not statically recoverable (doc 16) |
| `libvm.so` | string obfuscation uses **NEON** `movi/eor v16b` bulk XOR — **deobfuscated** (797 strings across two decoders; see `21-libvm-engine.md`) |

---

## 2. Virtual Master plugin overlays (decrypted)

The app ships four overlays as `assets/plugins/*.zip`, referenced by
`manifest.json` as `asset:///plugins/<name>.zip?h=true`. The `h=true` transform
is **AES-ECB with the same key `%z89aviCM0KkbEs9`** (Java
`Cipher.getInstance("AES")`), i.e. the sibling of the ROM's `e=x` XOR+7z. After
decryption they are plain zips:

| Overlay | Contents |
|---|---|
| `magisk.zip` (15.9 MB) | Magisk: `data/adb/magisk/{magisk64,magisk32,magiskinit64,magiskpolicy64,busybox64,busybox32,magiskboot64,…}`, `boot_patch.sh`, `util_functions.sh`, `chromeos/{futility,kernel.keyblock,kernel_data_key.vbprivk}`, `stock_boot.img`. |
| `superuser.zip` (1.45 MB) | `system/app/Superuser/Superuser.apk` (+odex), `system/xbin/daemonsu` — ChainsDD Superuser. |
| `xposed.zip` (4.55 MB) | `system/bin/app_process64_xposed`, `system/framework/XposedBridge.jar`, `system/lib64/libxposed_art.so`, `system/app/XposedInstaller/XposedInstaller.apk`, `system/xposed.prop`. |
| `play.zip` (101.9 MB) | `GoogleServicesFramework/`, `Phonesky/` (Play Store), `PrebuiltGmsCore/` (GMS). |

These are applied by the app's `z4.*` addon tasks (`MagiskTask`, `SuperuserTask`,
`XposedTask`, `GooglePlayTask`) which copy them into the guest's `system/` and
patch `init.rc` (see doc 04 §9).

### 2.1 Superuser overlay (root)

* `Superuser.apk` = `com.koushikdutta.superuser` v1.0.3.0 (label "Superuser").
* `daemonsu` = a static aarch64 ELF; strings: `/dev/socket/com.koushikdutta.superuser.daemon`,
  `/data/data/com.koushikdutta.superuser/databases/su.sqlite`,
  `/data/data/com.koushikdutta.superuser/files/multiuser_mode`,
  `Bad uid/gid %d/%d for Superuser Requestor application` — the classic ChainsDD
  Superuser `su` daemon (the same family twoyi's guest ships).

---

## 3. VMOS Pro assets

`vmospro.apk!/assets/` **[certain]**:

| Asset | Package / role |
|---|---|
| `libprelinker32.so`, `libprelinker64.so` | the guest prelinker interpreters (see doc 05/09) |
| `vmfwplugin` (JAR, `classes.dex`) | **framework plugin loader** — `com.android.framework.plugin.FrameworkPluginMain` (see §3.2) |
| `romex-framework-plugin.apk` | `com.vmos.frameworkplugin.romex` ("系统插件" / System Plugin) — the ROM framework plugin loaded by `vmfwplugin` |
| `romex_cn.apk` | `com.android.secex` ("RomEx") — a ROM extension |
| `vmos-pro-intent.apk` / `vmos-pro-intent-huawei.apk` | `com.vmos.pro.intent` ("launcher-intent") — launcher intent helper |
| `Launcher3.apk` | `com.android.launcher3` — the guest launcher |
| `pubkey` | a 128-hex-char (64-byte) key/hash |
| `siminfo.json` | a SIM/carrier database (MCC/MNC → operator, brand, bands) for the virtual telephony |
| `loading.json`, `anim_clean.json`, `h5_qr_back.png` | UI assets |
| `libwbsafeedit*` | Weibo SDK |
| `dexopt/baseline.prof` | baseline profile |

### 3.1 VMOS Pro plugins (root/overlays)

VMOS Pro bundles the same overlay families in `assets/plugins/`
(`magisk.zip`, `play.zip`, `superuser.zip`, `xposed.zip`), applied by its
`z4`-style tasks (see doc 05).

### 3.2 The framework plugin (`vmfwplugin` → `romex`)

`vmfwplugin` (`com.android.framework.plugin.FrameworkPluginMain`) implements the
hook interface `b`:
```java
onPreMakeApplication(String processName, Context ctx);
onAfterMakeApplication(String processName, Context ctx, Application app);
afterApplicationOnCreate(...);
afterApplicationTerminate();
```
The loader (`a`) scans **`/data/plugins/`** for plugin directories, reads
`conf.json` (`id`, `mainClassName`, `targetPackages`, `versionCode`), and for a
matching target package loads **`dexs/dexs.apk`** via `PathClassLoader` and
invokes `com.android.framework.plugin.api.MainLoadPackage.main(...)`
(`onPreMakeApplication`/`onAfterMakeApplication`). So it is a **per-app code
injection framework** in the guest — VMOS Pro's app-modification/multi-open
mechanism. The shipped `romex-framework-plugin.apk`
(`com.vmos.frameworkplugin.romex`) is the default such plugin.

---

### 3.3 The `romex` framework plugin — a full Xposed framework

`romex-framework-plugin.apk` (`com.vmos.frameworkplugin.romex`, label
"系统插件" / System Plugin) is a **complete Xposed framework** plus system-service
hooks and resource manipulation:

* **Xposed core**: `de.robv.android.xposed.*` — `IXposedHookZygoteInit`,
  `IXposedHookInitPackageResources`, `XC_MethodReplacement`, `MethodHooker`,
  `SELinuxHelper`, `DexCreator`, `PendingHooks`, `XC_LoadPackage`, `XCallback`.
* **System-service hooks** (`com.vmos.frameworkplugin.romex.system.*`) using
  `XposedHelpers.findAndHookMethod`:
  * `LocationManagerHandler` — hooks `com.android.server.LocationManagerService`
    (7 methods: `requestLocationUpdates`, `getBestProvider`, `getLastLocation`,
    `registerGnssStatusCallback`, `unregisterGnssStatusCallback`,
    `removeUpdates`, `addGnssMeasurementsListener`) → **virtual location**.
  * `VibratorHandler` — hooks `com.android.server.VibratorService` (6 methods:
    `vibrate`, `vibratePattern`, `hasVibrator`, `hasAmplitudeControl`, …) →
    **virtual vibrator**.
  * `WifiServiceHandler` — hooks `com.android.server.wifi.WifiServiceImpl`.
  * `PhoneHandler` — hooks `com.android.phone.PhoneInterfaceManager`
    (`getAllCellInfo`, `getCellLocation`, `getNeighboringCellInfo`) → **virtual cell**.
  * `SystemServerHandler` + `ExceptionHandler` — hooks
    `com.android.internal.os.RuntimeInit$LoggingHandler`/`UncaughtHandler`.
  * `GnssStatusListenerProxy`, `GnssStatusWrapper`, `LocationListenerProxy`,
    `MessageHandler` — the callback proxies.
* **Resource manipulation**: `pxb.android.axml`/`arsc` (AXML/ARSC parsers/writers)
  + `android.content.res.XResources`/`XResForwarder`, and a hook on
  `android.app.ApplicationPackageManager.getResourcesForApplication` — Xposed
  resource hooks.
* The plugin entry is `SystemLoadPackage` (a `MainLoadPackage`), which registers
  `SystemServerHandler`/`PhoneHandler`/`ExceptionHandler` per process.

So VMOS Pro injects an Xposed-based framework into the guest to virtualize
system services (location, vibrator, GNSS) and app resources.

### 3.4 The guest-side RomEx service (`com.android.secex`)

`romex_cn.apk` (`com.android.secex`, label "RomEx") is the **guest-side
counterpart** of the host's `ExEngineService`. Classes:

* `com.vmos.romex.{RomexApp, MainActivity, ExRomServiceBooter, Uitls}` —
  `RomexApp` loads native **`libvm_romex.so`**; `ExRomServiceBooter` (a
  `BroadcastReceiver`) starts the service via
  `ExRomClient.getInstance().connectExRomService(SampleRomService.class)`.
* `com.vmos.exsocket.*` — the **exsocket bus**: `ExRomClient`, `IRomService`,
  `ExConstants`, `ArgsParcel`, `plugin.{PluginSession,PluginSessionCtr}`,
  `utils.{LibcoreUtils, SystemApiUtils, ContextUserUtils, VmosCliper,
  MediaControllerHandler, RomNotificationListenerService}` and a shell
  (`utils.shell.*`).
* Native `libvm_romex.so` (arm64/armeabi) exports `JNI_OnLoad`, `jni_fstat`,
  `jni_readlink` (for `LibcoreUtils`).

`ExConstants` defines the guest-side protocol opcodes, e.g.
`EX_ENGINE_EXEC_CMD=46`, `EX_ENGINE_KEY_EVENT=48`, `EX_ENGINE_KEY_CODE=49`,
`EX_ENGINE_CREATE_USER=61`, `EX_ENGINE_QUERY_CONTENT_PROVIDER=44`,
`EX_ENGINE_GET_USER_IDS=59`, `EX_ENGINE_INVOKE_SERVICE_METHOD=53`,
`EX_ENGINE_PROCESS_MEMORY_INFO=47`, plus notification/volume/file verbs and the
actions `com.vmos.exsocket.engine.ACTION_*`. This is the **guest⇄host ExEngine
protocol** (the host side is `com.vmos.exsocket.engine.ExEngineService`).



## 4. The Virtual Master guest control plane: `VMServer` (`vmserver-service.jar`)

The Virtual Master guest ships `system/framework/vmserver-service.jar` (43 KB,
added to `SYSTEMSERVERCLASSPATH`), which loads a **`SystemService`** into the
guest's `system_server`:

```
com.android.server.vmserver.VMServer          (extends android.server.SystemService)
com.android.server.vmserver.VMEventManager
com.android.server.vmserver.VMEvents          (the 22 action constants)
com.android.server.vmserver.IVMEventCallback
com.android.server.vmserver.LocalServerSocket (reflection on LocalSocketImpl)
com.android.server.vmserver.ReflectUtils
```

### 4.1 Transport

`VMServer.onStart()` creates `new VMEventManager(ctx, false, "/dev/vm_event")`
— **client mode**, **FILESYSTEM namespace**, socket **`/dev/vm_event`** (the host
is the server, binding the socket the guest sees as `/dev/vm_event`). Framing is
`DataInputStream.readUTF()` / `DataOutputStream.writeUTF()` with the payload
**``<event>`<data>``** (backtick separator) — identical to the host app's
`dev/event` protocol. `LocalServerSocket` uses reflection on
`android.net.LocalSocketImpl`.

### 4.2 Guest→host events (`sendEvent`)

`VM_SERVER_READY`, `ANDROID_OS_READY`, `BOOT_COMPLETED`, `TASK_STACK_CHANGED`
(via an `ITaskStackListener`), `INSTALL_APP_RESULT`,
`UNINSTALL_APP_RESULT`, `CLIPBOARD_DATA`. System broadcasts are forwarded by
`VMEventManager.registerForSystemEvents`: `PACKAGE_ADDED`/`PACKAGE_REMOVED`
(`pkg path launcherable`), `CONFIGURATION_CHANGED` (`{"orientation":N}`),
`OPEN_OUTER_PAGE`, `EXECUTE_COMMAND`.

### 4.3 Host→guest commands (`onEvent`)

| Action | Effect in the guest |
|---|---|
| `SET_LOCALE_AND_TIME` | `LocalePicker` + `persist.sys.timezone` + 12/24h |
| `CLIPBOARD_DATA` | set the guest clipboard |
| `OPEN_OUTER_PAGE` | `startActivity` (return to the host UI) |
| `SYNC_PROP_EVENT` | `SystemProperties.set(key, value)` |
| `SEND_KEY_EVENT` | `InputManager.injectInputEvent` |
| `START_INSTALL_APP` | `PackageInstaller` session (from the host-provided APK paths) |
| `START_UNINSTALL_APP` | `PackageManager.deletePackage` |
| `KILL_APP` | kill the package |
| navigation/recents/notifications actions | `StatusBarManager`/`ActivityManager` |

`VMServer` also exposes `getShellAppApkPath()` (reads `VM_SHELL_APK_PATH`, the
host app's APK path inside the guest) and `getApkCertMd5()` (APK signature MD5).

This is the full **guest⇄host control plane** — a system service (not just the
`com.android.vmservice` app) driving app install, keys, clipboard, locale,
navigation and the task stack.

---

## 5. What this adds to the picture

* The Virtual Master host⇄guest control plane is a **system-service + app** pair
  (`vmserver-service.jar` + `com.android.vmservice`) talking over
  **`/dev/vm_event`** with UTF/backtick framing — the guest-side counterpart of
  the host app's `<vmRoot>/dev/event` bus.
* The overlay mechanism is **AES-ECB** (`?h=true`) for the plugin zips and
  **XOR+7z** (`?e=x`) for the ROM images — the same key `%z89aviCM0KkbEs9`.
* VMOS Pro adds a **framework plugin loader** (`vmfwplugin` + `/data/plugins/` +
  `romex`) for per-app code injection, plus guest-side assets (`Launcher3`,
  `romex`, `vmos-pro-intent`, `siminfo.json`, `pubkey`).
* Root is delivered by **overlays** (Magisk/Superuser/Xposed/Play), not baked
  into the base image — for both Virtual Master and VMOS Pro.
