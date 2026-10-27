# 04 — Virtual Master (`com.clone.android.dual.space`)

> **Naming:** this APK's app label is **"Virtual Master"** (package
> `com.clone.android.dual.space`). Virtual Master is a **distinct product from
> VMOS** (a different company) — it is *not* the same app, despite the
> `com.clone` package name and the shared ROM-format lineage. The
> `workspace/apk/virtualmaster` / `workspace/jadx/virtualmaster` directories and
> the `virtualmaster_*` notes are Virtual Master's (an earlier `vmos` prefix was
> a mislabel). See `19-virtual-master-and-rom.md` for its decrypted ROM.

Virtual Master is a closed-source commercial "virtual machine". Structurally it is the
same class of thing as twoyi (an Android userspace container on the host kernel),
but implemented inside native code (`libvm.so`) with a different bootstrap chain,
an ordered task pipeline, and a privileged activation path.

Sources: `subjects/VM-v3.2.80-official.apk`; decompiled at `workspace/jadx/virtualmaster/`;
native analysis in `workspace/notes/virtualmaster_core.txt`; **fully deobfuscated Java
report in `workspace/notes/virtualmaster_java_deobfuscated.md`** (all strings below are
from the deobfuscator `workspace/notes/virtualmaster_deobf.py`).

---

## 1. Native components

| Library | Size | Role |
|---------|------|------|
| `libvm.so` | 9.7 MB | The VM engine. OLLVM-obfuscated; its strings are XOR-per-string and were **deobfuscated** (540 strings) — see `21-libvm-engine.md`. |
| `libkrloader64.so` / `libkrloader32.so` | 234 KB | **Host-side loader** (INTERP `/system/bin/linker64`); obfuscated. |
| `libkr64.so` / `libkr32*.so` | 1.6 MB | **Guest dynamic linker** = patched bionic `linker64` **+ ShadowHook 2.0.1**. |
| `libun7z.so` | 76 KB | 7-Zip extraction. |
| `assets/plugins/{magisk,superuser,xposed,play}.zip` | — | Root/Xposed/Play overlays. |

See doc 09 for the loader chain and doc 16 for the obfuscation.

---

## 2. Bootstrap chain

```
VMApp  → System.loadLibrary("vm")            [log: "[init] loading vm library"]
  └─ NativeHelper.startOS(vmid, osVersion, romVersion, <dataDir>/lib64)
       ├─ BuildExecPathTask: <dataDir>/lib64 → applicationInfo.nativeLibraryDir
       └─ libkrloader64.so            INTERP /system/bin/linker64
            ▼
          libkr64.so                  INTERP <dataDir>/lib64/libkrloader64.so
            │  patched bionic linker + ShadowHook (hooks soinfo ctors/dtors)
            ▼
          guest init → zygote64 → system_server → apps
```

`libkr64.so` is the "kernel": a bionic `linker64` that (a) loads the guest and
(b) provides **inline hooking** (`libshadowhook.so`, "hook (with island) OK") by
hooking `soinfo::call_constructors`/`call_destructors`, after scanning linker
memory to find the right offsets. This is Virtual Master's advertised Java+native Hook
capability. See `09-native-runtime-and-loaders.md` §3.

---

## 3. The orchestration pipeline (`q4.WoWo`)

`WoWo` is the per-VM state machine. It runs three ordered phases then boots.

### Phase 1 — fs-prepare (state 2)

| Task (`getName()`) | Purpose |
|---|---|
| `PrepareFsTask` | mkdir `<vmRoot>/fs`, `<vmRoot>/fs/dev`; chmod |
| `InstallFsTask` | install the fs subtree |
| `FixFsTask` | rename `_10`/`_51` shims back (see below) |
| `CleanFsTask` | delete stale `init_10`, `kr64_*`, `libui*.so`, ranchu HALs, `Launcher3QuickStep*` |
| `ChmodFsTask` | recursive chmod |
| `CleanCacheTask` | clear `/data/dalvik-cache` |
| `FixCPUArchTask` | rewrite `ro.zygote` (`zygote64_32`/`zygote64`) and ABI lines |
| `LoadVMPropTask` | load `ro.product.*` from `/system/build.prop` |

`FixFsTask` exact rename pairs: `/init_10`→`/init`, `/sbin/adbd_10`→`/sbin/adbd`,
`/system/bin/sh_10`→`/system/bin/sh`, `libui10.so`/`libui51.so`→`libui.so`,
`libhostlibui_10.so`→`libhostlibui.so` (lib and lib64). This is how Virtual Master
swaps per-guest-version UI/linker shims at install time.

### Phase 2 — host services up (`[doStartSvc]`), **before** boot

In order (with decoded log tags and error offsets):

```
[setupBinder] start binder    → BinderService.setupBinder(vmid)      (+114000)
[setupInput]  start input     → InputService.nativeSetup(...)        (+110000)
[setupAudio]  start audio     → AudioService.start()                 (+111000)
[setupHAL]    start hal       → HALManager.startHALMgr()             (+112000)
[setupRender] start render    → DisplayService.nativeSetup(w,h)      (+109000)
[setupNetlink]start netlink   → NetlinkManager.start()               (+112500)
[setupVpn]    start vpn       → VpnManager.start()
[setupBridge] start bridge    → register WoWo as event listener
```

### Phase 3 — addon/root install (state 4)

`ApplyOverlaysTask, Bug1..8FixTask, CleanLogTask, SuperuserTask, XposedTask,
GooglePlayTask, MagiskTask, BuildTmpfsTask, BuildVMPropTask, BuildExecPathTask`.

Then:

```java
KLog("[doStartOS] start os");
int pid = NativeHelper.startOS(vmid, os_version, rom_version, lib64);
```

On success state 5 + health polling; on failure state −4, error `(-ret)+117000`.

### Per-VM layout

```
<dataDir>/vm/vm<N>/
  fs/                      guest rootfs
  fs/dev/                  guest /dev
  dev/event                host↔guest event socket (LocalSocket, FILESYSTEM)
  dev/input/touch          input node
  dev/tun.ctrl_client      VPN control
  dev/netlink_client       netlink config
<dataDir>/lib64 → nativeLibraryDir     (holds libkrloader64.so, libkr64.so)
```

`BuildTmpfsTask` also creates `/dev/socket`, `/dev/vmproc`, `/dev/__properties__`,
`/dev/__kmsg__`, `/dev/__kmsg2__`, `/dev/kmsg`, `/dev/dm-user` and tweaks
`kptr_restrict`, `mmap_rnd_bits`.

---

## 4. Guest OS identity

ranchu/marlin-based Android, multiple versions:
`vendor/etc/init/hw/init.marlin.rc`, `init.ranchu.rc`, `fstab.ranchu`,
`vendor/lib64/hw/{camera.ranchu, camera.ranchu.jpeg, vulkan.ranchu,
gralloc.ranchu, vibrator.ranchu, audio.primary.ranchu,
audio.primary.ranchu_legacy, fingerprint.ranchu,
android.hardware.sensors@2.1-impl.ranchu}.so`,
`vendor/bin/hw/android.hardware.gnss@2.0-service.ranchu`,
`vendor/lib64/libqemupipe.ranchu.so`, `system/bin/kr64_10/kr64_81/kr32_10/kr32_81`.

> Note `libqemupipe.ranchu.so`: Virtual Master keeps the **QEMU-pipe HAL** for some
> services, unlike twoyi which replaced it wholesale.

---

## 5. Event protocol (fully decoded)

* Channel: Unix `LocalSocket`, namespace FILESYSTEM, path `<vmRoot>/dev/event`
  (server binds + `listen(50)`; client loops reconnect).
* Framing: `DataInputStream.readUTF()` / `writeUTF()`, payload
  **`<action>`<arg>`** — backtick delimiter.
* 22 actions (`com.android.vmcore.action.*`):

```
VM_SERVER_READY, ANDROID_OS_READY, BOOT_COMPLETED,
START_INSTALL_APP, INSTALL_APP_RESULT, START_UNINSTALL_APP, UNINSTALL_APP_RESULT,
TASK_STACK_CHANGED, CLIPBOARD_DATA, OPEN_OUTER_PAGE, TOGGLE_RECENTS,
SEND_KEY_EVENT, SET_LOCALE_AND_TIME, SET_GLOBAL_SETTING,
SHOW_NAVIGATION_EVENT, HIDE_NAVIGATION_EVENT, SYNC_PROP_EVENT, KILL_APP,
EXPAND_NOTIFICATIONS_PANEL, COLLAPSE_NOTIFICATIONS_PANEL,
SET_NAVIGATION_BAR_RTL, EXECUTE_COMMAND
```

Argument encodings: `EXECUTE_COMMAND` = JSON `{"pkg","cmd","arg"}` with `cmd` ∈
`notification.post/remove/removeAll`, `clipboard.set_clipboard_data`;
`SET_LOCALE_AND_TIME` = `"<lang> <country> <variant> <12|24> <tzid>"`;
`SET_GLOBAL_SETTING` = `"captive_portal_mode 0"`, etc.; `SYNC_PROP_EVENT` =
`{"key","value"}`; `BOOT_COMPLETED` (in `VMExtension`) = `{"orientation":N}`.

---

## 6. Binder bridge (exact mechanism)

`BinderService.m5106(vmApp, vmid)`:

1. `intent.setComponent(packageName, "com.android.vmcore.service.BinderService")`.
2. **Obtain the `peekService` transaction code by reflection**:
   `ServiceManager.getService("activity")` → `IActivityManager$Stub.asInterface`,
   then `ActivityManager.IActivityManagerSingleton/mInstance` (SDK≥26) or
   `ActivityManagerNative.gDefault` (older), with an `InvocationHandler` that
   captures the int returned by `peekService`.
3. Build a Parcel: `writeInterfaceToken("android.app.IActivityManager")`,
   `writeInt(1)`, `intent.writeToParcel`, `writeString(null)`, `writeString(pkg)`.
4. `setupBinder(vmid, peek, 1, 2, "com.android.vmcore.service.IBinderService",
   parcel.marshall())` (native) — injects the transaction into the guest.
5. `bindService(intent, conn, 1)` with a 5 s latch; logs `[publish] service bind ok`
   / `timeout` (−502) / `can not get peek code` (−501).

---

## 7. Display, input, HAL

* **Display** (`DisplayService` JNI): `nativeSetup(vmid,osVer,romVer)`,
  `nativeAddSurface(ptr,id,Surface,w,h,scale)`, `nativeRemoveSurface`,
  `nativeStartService/StopService`, `nativeGetFPS`, `nativeDispose`.
  `VMSurfaceView` manages a `SurfaceView`, computes letterbox/orientation in
  `onMeasure`, and passes a scale for touch mapping.
* **Input** (`InputService` JNI): `nativeSetup(vmid, flag)` (flag =
  `os_version>=11 && !support_a32`), `nativeOnTouchEvent(ptr,action,pointerId,
  timeNanos,x,y)`; the guest node is `<vmRoot>/dev/input/touch`.
* **HALManager** — camera (list/frame/focus/zoom/picture), location, and a full
  **telephony AT-command engine** (`ExecPhoneCommand`, commands split on `;`,
  handlers `+CREG +CSQ +COPS +CGMR +CGSN +CRSM +CIMI +CFUN +CPIN +CLCC +CMGS …`).
* **BatteryService** — emits `BATTERY_CHANGED`, sysfs
  `/dev/hal/power_supply/{battery,dc,usb}` + `/dev/hal/power_update`.
* **AudioService** — `acquireAudioRecord/Track` (needs `RECORD_AUDIO`).
* **NetlinkManager** — host Wi-Fi/cell netlink → guest; defaults MAC
  `02:15:b2:00:00:00`, SSID `AndroidWifi`, ifname `wlan0`.
* **VPN** — `VMVpnService` (VpnService) + `VpnManager`/`VpnTunService` native TUN
  relay; `VMNetworkConfig` = `ifname/mac/ip/network/netmask/gateway_ip/
  ipv6_prefix/dns4_ip/dns6_ip`.

---

## 8. ROM catalogue & install

* Built-in ROMs (`n3.C3990`): `rom_4.2.2`, `rom_5.1.1`, `rom_7.1.2(_32)`,
  `rom_9.0.0`, `rom_11.0.0` — served as **Play Asset Delivery** packs
  (`pad://rom_<v>`).
* Remote list: Firebase Remote Config key `rom_configs_v3280`; channels
  `officialmini`/`officialfull`.
* Import: copy to `<externalCacheDir>/tmp/rom_<id>` → md5 verify against every
  `Asset.md5` → read `manifest.json` → unzip (blankj `ZipUtils`; `.7z` via
  `libun7z`) to `<externalFilesDir>/rom/<id>_<ver>` → rename.
* Package members: `vm_config.xml`, `rom.zip`, `superuser.zip`, `xposed.zip`,
  `play.zip`, `magisk.zip`, `data.zip`.
* Asset rewriting: `asset:///rom/rom_7_1_2/{magisk,superuser,play}.zip` →
  `asset:///plugins/...`.

---

## 9. Root/overlay injection (guest files)

* **Magisk** (`MagiskTask`): `/sbin/{magisk,magisk32,magisk64,magiskpolicy,
  resetprop,supolicy,su,stub.apk,com.topjohnwu.magisk.apk}`,
  `/sbin/.magisk/{block,mirror,worker,config}`,
  `/data/adb/magisk/{busybox64,magiskboot64,magiskinit64,magiskpolicy64,…}`,
  `/data/adb/{magisk.db,modules,post-fs-data.d,service.d}`; patches `init.rc`
  with the `# magisk` / `# magisk start` blocks (service `pfdsvc`/`lssvc`,
  `on property:sys.boot_completed=1`, `on property:init.svc.zygote=…`).
* **Superuser** (`SuperuserTask`): `/system/app/Superuser/Superuser.apk`,
  `/system/xbin/daemonsu`, `/system/bin/su`, `/system/xbin/su`,
  `/dev/socket/com.koushikdutta.superuser.daemon`; injects
  `service daemonsu /system/xbin/daemonsu --daemon`.
* **Xposed** (`XposedTask`): `/system/bin/app_process{32,64}_xposed`,
  `libxposed_art.so`, `/system/framework/XposedBridge.jar`.
* **Google Play** (`GooglePlayTask`): `/system/priv-app/{Phonesky,
  PrebuiltGmsCore,GoogleServicesFramework}`.
* **Bug fix tasks**: `/data/property/persist.sys.root_access=3`,
  `/data/misc/shared_relro/`, Huawei hwid path, `/rm_debug.txt`,
  `DELETE FROM secure WHERE name='android_id'`, etc.

---

## 10. Activation (Android 14+ phantom-process killer)

The real reason for activation is now known: **Android 14+ caps "phantom"
(child) processes**, and a container needs many (zygote, system_server, apps).
`DisablerLauncher.isActivationRequired()` gates on SDK≥31. Three methods:
`shizuku`, `wireless_adb`, `direct`.

Wireless-ADB flow:
* mDNS `_adb-tls-pairing._tcp.`; default host `127.0.0.1`, connection port
  **5555**; 6-digit pairing code.
* `AdbConnManager` generates an RSA-2048 key + self-signed X.509 (`CN=host`,
  2-year), cached in the app files dir (`adb_client_keypair.dat`, …).
* ADB CNXN payload `host::\u0000`, TLS exporter label `adb-label\0`, 24-byte LE
  packet header.
* Unlock pipeline (over ADB shell):
  ```
  device_config set_sync_disabled_for_tests persistent
  device_config put activity_manager max_phantom_processes 2147483647
  settings put global settings_enable_monitor_phantom_procs false
  device_config get activity_manager max_phantom_processes
  ```
* On rooted devices, `su` is used instead.

See `14-activation-privilege-and-security.md`.

---

## 11. Telemetry / misc

* `VMApp`: Firebase project `virtualmaster`; registers package add/remove
  receiver.
* Report builder (`n4.WWWW`): device/mem/CPU/GPU info, `uname -r`, `which su`,
  logcat dumps, and encrypts `vminfo.txt` with **AES-ECB/PKCS5 using the
  hard-coded key `%z89aviCM0KkbEs9`** (16 bytes).
* API base **`https://api.virtualmaster.app/account/v1`**; web
  `virtualmaster.app`. Virtual Master is operated by "VirtualMaster".
* Analytics: `vm_boot_failed`, `vm_boot_time`.

---

## 12. Comparison anchors

* Shares with twoyi: goldfish/ranchu guest, custom loader + guest linker,
  emulator graphics stack, socket transport, HAL/input virtualization, rootfs as
  a directory.
* Differs: one obfuscated `libvm.so`; loader chain `libkrloader64` + `libkr64`
  (patched bionic linker + ShadowHook); an ordered task pipeline; guest-version
  UI shims; activation to defeat the phantom-process killer; telemetry with a
  hard-coded AES key.
