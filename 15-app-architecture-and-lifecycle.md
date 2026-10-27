# 15 — App Architecture & Lifecycle (deep dive)

The host app is not just a launcher — it is the container runtime's control
plane. This doc maps the app layer of each project: components, state machines,
and the end-to-end lifecycle (first boot, normal boot, reboot, shutdown,
install, file exchange).

---

## 1. twoyi app

### 1.1 Components (from source + manifest)

| Component | Role |
|-----------|------|
| `TwoyiApplication` | `attachBaseContext` → `RomManager.ensureBootFiles()` + `TwoyiSocketServer.start()`; AppCenter analytics in `onCreate` |
| `Render2Activity` | full-screen `SurfaceView`; boot UI (spinner + boot-log texture); `Renderer.init/resetWindow/removeWindow`; touch + back-key forwarding |
| `Renderer` (JNI) | `libtwoyi.so` façade |
| `TwoyiStatusManager` | singleton; `mStarted` + `CyclicBarrier` boot latch; `switchOs` toggles guest/home |
| `TwoyiSocketServer` / `TwoyiMessenger` | abstract `TWOYI_SOCK` control channel |
| `RomManager` | ROM extract/upgrade/reset, rootfs/vendor dirs, loader symlinks, orphan kill, last-kmsg |
| `Installer` | drives `libadb.so` to install APKs into the guest (port 22122) |
| `SelectAppActivity` | pick APKs (`ACTION_GET_CONTENT`) and install into the guest |
| `SettingsActivity` | replace ROM (`ACTION_GET_CONTENT` zip), export log, About, shortcuts |
| `TwoyiDocumentsProvider` | `DocumentsProvider` exposing the guest `sdcard` for file exchange |
| `BootLogTexture` | streams guest `logcat -v brief *I` into a `TextureView` (colored by level) |
| `UIHelper` / `UITips` | dialogs, links, Android-12 tips |
| `LogEvents` / `ACache` / `CacheManager` / `AppKV` | logging, caching, preferences |

### 1.2 Lifecycle

```
App start
  TwoyiApplication.attachBaseContext
     ├─ RomManager.ensureBootFiles()      create <rootfs>/dev/{input,socket,maps},
     │                                    <dataDir>/socket, loader64 symlink, kill orphans
     └─ TwoyiSocketServer.start()         bind abstract TWOYI_SOCK
  Render2Activity.onCreate
     ├─ reset status; hide nav
     ├─ UITips.checkForAndroid12 → bootSystem()
     └─ SurfaceView callback → Renderer.init(surface, loaderPath, dpi, fps)
             ├─ host: startOpenGLRenderer + exec <rootfs>/init (TYLOADER=loader64)
             └─ guest boots; sends BOOT_COMPLETED → boot latch released
  boot UI hidden; guest UI shown
  Back key → KEYCODE_HOME to guest; SWITCH_HOST toggles host launcher
  Reboot → finish + RomManager.reboot (relaunch intent + System.exit)
  Factory reset / replace ROM → extractRootfs (p7zip) + initRootfs
```

Notable behaviors:
* `onCreate` while `isStarted()` → **reboot** (state is not restorable).
* `onRestoreInstanceState` → reboot (no state restore supported).
* Guest → host `SETTINGS` opens the host settings.

---

## 2. Virtual Master app

### 2.1 Components

| Component | Role |
|-----------|------|
| `VMApp` (`com.android.vmapp.VMApp`) | application; `System.loadLibrary("vm")`; state init |
| `q4.WoWo` | per-VM **state machine** (validate → fs-prepare → addons → `startOS` → services) |
| `n4/s4` | VM monitor + event manager |
| `VMCoreService` / `BinderService` | foreground service + binder bridge |
| `VMVpnService` | network |
| `VMDisplayActivity` | the guest display `SurfaceView` |
| `VMStartActivity0..3` | boot/health UI (shows `DisablerLauncher.isActivationRequired`) |
| `VMStopActivity` / `VMReportActivity` | stop / diagnostics |
| `MainActivity` | launcher |
| `com.android.vmcore.hal.*` | `DisplayService`, `AudioService`, `NetlinkManager` |
| `com.android.vmcore.vpn.*` | `VpnManager`, `VpnTunService` |
| `com.phantom.killer.disabler` | ADB/Shizuku activator |

### 2.2 Lifecycle

```
VMApp → load libvm.so
  WoWo.validate → fs-prepare tasks (y4/) → addon tasks (z4/)
     ├─ BuildExecPathTask: <dataDir>/lib64 → nativeLibraryDir
     ├─ Superuser/Magisk/Xposed/GooglePlay tasks copy plugin zips into the guest
     └─ device-node task: /dev/{socket,vmproc,__properties__,kmsg,dm-user}
  NativeHelper.startOS(vmid, osVer, romVer, <dataDir>/lib64)
  setup services: binder, input, audio, HAL, render, netlink, vpn, bridge
  event bus (<vmDir>/dev/event) reports BOOT_COMPLETED
  UI switches to VMDisplayActivity
```

---

## 3. VMOS Pro app

### 3.1 Components

| Component | Role |
|-----------|------|
| `PureApplication` | application |
| `PureSplashActivity` / `MainActivity` | launcher; `MainActivity` handles `rom://com.vmos.pro` |
| `com.vmos.assistant.ui.*` | `AssistantMainActivity`, `ActivationPrepareActivity`, `AboutActivity` |
| `WirelessDebugPairService` / `ReceivePairCodeInputService` | pairing flow |
| `CoreAccessibilityService` | auto-enable wireless debugging |
| `Virtual MasterEngineServiceFactory$VMS<hex>` | per-VM engine service (`:vms<hex>`, action `com.vmos.core.ENGINE_SERVICE`) |
| `HWServiceFactory$HWS1..100` | 100 delegated-zygote hardware services (`:m1..:m100`) |
| `ExEngineService` | `osimg/socket/` bus (`ex_engine`/`ex_plugin`) |
| `com.vmos.core.hwservice.*` | HAL/zygote virtualization |
| `KeepLiveVpnService`, `DetectPhantomProcessReceive` | keep-alive, process detection |
| `com.vmos.pro.aidlcommunicate` | window-service AIDL protocol (`UPDATE_VM_STATUS`, `CLOSE_WINDOW_SURFACE`, …) |

### 3.2 Lifecycle

```
PureSplashActivity → first-run init of /osimg/r, config/envinfo migrations
  → activation (wireless ADB → helper → Netty 22681)
  → per-VM: unpack ROM into osimg/r/otNN, launch engine service (:vmsXX)
       → HW services (:mN) call NativeUtil.startzygote_d*
       → libkrnloader --file libvmoskernel.so → guest kernel → guest init
       → renderer (libvmtools) + input + audio + HAL
  → ExEngineService bus for app install/file transfer
```

---

## 4. Cross-project lifecycle comparison

| Phase | twoyi | Virtual Master | VMOS Pro |
|-------|-------|------|----------|
| First-run | extract `rootfs.7z` | download/unzip ROM | download/unzip + decrypt ROM |
| Setup | `ensureBootFiles` + loader symlink | `lib64` symlink + addon tasks | per-VM `osimg` tree + prelinker |
| Privilege | none | Shizuku/ADB/su | wireless ADB + helper (22681) |
| Boot | exec static `init` | `startOS` | `libkrnloader` → guest kernel |
| Ready signal | `BOOT_COMPLETED` (abstract socket) | `ANDROID_OS_READY`/`BOOT_COMPLETED` (event) | engine ready + bus |
| Display | `SurfaceView` + `libOpenglRender` | `VMDisplayActivity` + `libvm.so` | `libvmtools` + `Surface` |
| Install app | `libadb.so` → guest adbd:22122 | event `START_INSTALL_APP` | `EngineSession` + FD passing |
| File exchange | `DocumentsProvider` | guest adb / file UI | `EngineSession` transfer |
| Reboot/reset | relaunch + `System.exit` | VM stop/start | VM stop/start |

---

## 5. Notes for building the app layer

* Keep the **runtime control plane** separate from the UI (a state machine with
  explicit states: `Idle → Extracting → Booting → Running → Stopping`).
* Use a **single, versioned control protocol** (not several ad-hoc sockets) and
  make it **authenticated**.
* Make boot observable (log stream, like twoyi's `BootLogTexture`) — it makes
  failures debuggable and the UX far better.
* Expose file exchange through a `DocumentsProvider` (twoyi) rather than
  ad-hoc paths.
* Treat reboot as "stop + start", and never rely on `Activity` state restore.
* Support **multiple profiles/VMs** from day one (per-VM rootfs, ports, mode) —
  twoyi's fork threetwi added this later; VMOS Pro has it.
