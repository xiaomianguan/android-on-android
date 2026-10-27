# 17 — End-to-End Boot Sequence (all three, step by step)

The single most useful reference when implementing: what *actually happens*, in
order, from tapping the icon to seeing the guest launcher. Every step names the
concrete component, syscall, socket or command.

---

## 1. twoyi

```
① App process start
   TwoyiApplication.attachBaseContext
     • RomManager.ensureBootFiles()
         - mkdir <rootfs>/dev/{input,socket,maps}, <dataDir>/socket
         - symlink <dataDir>/loader64 → libloader.so
         - kill orphaned children of PID 1 (ps -ef | awk '$3==1' | xargs kill -9)
     • TwoyiSocketServer.start()  → bind abstract LocalSocket "TWOYI_SOCK"

② Activity + surface
   Render2Activity.onCreate
     • TwoyiStatusManager.reset()
     • setContentView(ac_render); create SurfaceView
     • UITips.checkForAndroid12 → bootSystem()
   bootSystem():
     • if !romExist || upgrade → new Thread "extract-rom":
         RomManager.extractRootfs()  (7z x rootfs.7z -o<dataDir>)
         RomManager.initRootfs()     (write vendor/default.prop)
     • add SurfaceView; showBootingProcedure()
   surfaceCreated():
     • Renderer.init(surface, getLoaderPath(), xdpi, ydpi, fps)

③ Native boot (libtwoyi.so renderer_init)
     • ANativeWindow_fromSurface(surface)
     • renderer_bindings::startOpenGLRenderer(win,w,h,xdpi,ydpi,fps)
         → libOpenglRender.so starts; listens on rootfs/opengles{,2,3}
     • Command::new("./init")
         .current_dir("/data/data/io.twoyi/rootfs")
         .env("TYLOADER", <libloader.so>)
         .spawn()

④ Guest bootstrap (static init)
     • init reads TYLOADER; establishes the virtual root
       (all later processes use patched libc/linker64 `translate_path`)
     • init runs Android init: ueventd, (hw/vnd)servicemanager, zygote64 …
     • guest /dev is materialized as sockets: dev/socket/property_service,
       dev/input/{touch,key0} (served by libtwoyi.so), dev/socket/…
     • binder: libbinder/libvbinder (userspace driver over sockets)

⑤ Display + input come alive
     • SurfaceFlinger → HIDL composer@2.1 → hwcomposer.twoyi.so
         → HostConnection/rcEncoder → socket rootfs/opengles → libOpenglRender
     • guest EventHub reads dev/input/touch (input.rs streams input_event)
     • app MotionEvent → Renderer.handleTouch → ABS_MT_* events

⑥ Ready
     • guest init writes property; guest process sends "BOOT_COMPLETED"
       over TWOYI_SOCK
     • TwoyiStatusManager.markStarted() releases the boot latch
     • UI hides the boot log and shows the guest

⑦ Steady state
     • Back key → sendKeycode(KEYCODE_HOME)
     • SWITCH_HOST → toggle host launcher / guest
     • install APK: Installer → libadb.so → adb connect localhost:22122 →
       install -t -r
     • file exchange: TwoyiDocumentsProvider
```

---

## 2. Virtual Master

```
① App process start
   VMApp: System.loadLibrary("vm")  → libvm.so   ("[init] loading vm library")
   VMStartActivity0 → VMStartActivity{1,2,3} by VM kind
     → q4.WoWo.m18178(true)   (the per-VM state machine)

② Phase 1 — fs-prepare (state 2)
   PrepareFsTask, InstallFsTask, FixFsTask (rename _10/_51 shims),
   CleanFsTask, ChmodFsTask, CleanCacheTask, FixCPUArchTask, LoadVMPropTask

③ Phase 2 — host services up  ([doStartSvc])
   BinderService.setupBinder  (reflection: peekService code; Parcel
       "android.app.IActivityManager"; native setupBinder(...))
   InputService, AudioService, HALManager, DisplayService, NetlinkManager,
   VpnManager, bridge listener

④ Phase 3 — addon/root install (state 4)
   ApplyOverlaysTask, Bug1..8FixTask, CleanLogTask, SuperuserTask, XposedTask,
   GooglePlayTask, MagiskTask, BuildTmpfsTask (creates /dev/__properties__,
   /dev/kmsg, /dev/vmproc, …), BuildVMPropTask, BuildExecPathTask
       → <dataDir>/lib64 → nativeLibraryDir

⑤ Boot
   NativeHelper.startOS(vmid, os_version, rom_version, <dataDir>/lib64)
     → libvm.so execs the guest via:
         libkrloader64.so (host loader)
           → libkr64.so  (patched bionic linker + ShadowHook)
             → guest init → zygote64 → system_server → apps

⑥ Display + input + HAL
   DisplayService.nativeAddSurface(VMSurfaceView surface) → native renderer
   InputService.nativeOnTouchEvent(...) → <vmRoot>/dev/input/touch
   HALManager (camera/location/telephony), AudioService, BatteryService
   NetlinkManager → /dev/netlink_client ; VpnManager → /dev/tun.ctrl_client

⑦ Ready
   guest sends com.android.vmcore.action.BOOT_COMPLETED / ANDROID_OS_READY
   over <vmRoot>/dev/event (UTF, backtick-framed)
   VMHealthMonitor polls init/zygote/system_server; UI shows the VM

⑧ Steady state
   event actions for install/uninstall/clipboard/keys/nav; VPN TUN relay
```

---

## 3. VMOS Pro

```
① App start
   PureApplication; PureSplashActivity → first-run /osimg init + migrations
   activation gate: ActivationPrepareActivity → WirelessDebugPairService
     (mDNS _adb-tls-pairing._tcp., 6-digit code, Shizuku AdbPairingClient)
   → AdbShellCmdTool:
        cp libvmos.so        /data/local/tmp/vmos_starter
        cp vmos_start_sh.so  /data/local/tmp/vmos_starter_sh.sh
        sh -c /data/local/tmp/vmos_starter_sh.sh
   → libvmos.so becomes "com.vmos.helper_service"; Netty server on TCP 22681

② Per-VM setup
   ROM → <dataDir>/osimg/r/otNN/{rootfs,guestOSInfo,libvmoskernel.so,…}
   engine service :vmsXX (action com.vmos.core.ENGINE_SERVICE)
   HW services :m1..:m100 (NativeUtil.startzygote_d*)  ← phantom-process workaround

③ Boot
   libkrnloader.so --file <vmDir>/libvmoskernel.so
     → guest "kernel" → guest init
         interpreter → libinitlinker64.so / libylinker64.so
           INTERP → /data/data/com.vmos.pro/vmb/libprelinker64.so
             (bionic linker + seccomp + sigsegv handler)
               → guest bionic userspace

④ Display/input/audio/HAL
   libvmtools.so / libvmkrntools.so (emulator renderer):
     vmtools_init, vmtools_resetup_window(Surface), vmtools_input_event64,
     vmtools_getFrameBufferRGBA ; sockets rfopenglesNNN / NATIVEBUFFERsocket
   libhalbox.so (camera/GPS/WiFi/sensors/GSM)
   NativeAudioManagerUtil (rfaudio socket + PCM bridge)
   libmci.so media/control protocol (FFmpeg, handshakeEncrypt)

⑤ Ready / bus
   ExEngineService: osimg/socket/{ex_engine,ex_plugin}
     (length-prefixed Parcel frames + FD passing) for install/file/intent ops
```

---

## 4. Where each system spends its "startup budget"

| Step | twoyi | Virtual Master | VMOS Pro |
|------|-------|------|----------|
| ROM extract | 7z, one-time | unzip/decrypt, one-time | unzip/decrypt, one-time |
| Pre-boot | `ensureBootFiles` | 8+ fs tasks, 8+ addon tasks | per-VM osimg + engine services |
| Privilege | none | ADB/Shizuku | wireless ADB + helper |
| Boot exec | static init | loader→linker | kernel loader→prelinker |
| Ready signal | abstract socket | event LocalSocket | engine bus |

---

## 5. Failure points to watch (from the code)

* **Extraction**: bad md5, wrong ROM format, insufficient space.
* **Privilege**: activation not done (Virtual Master/VMOS Pro), phantom-process killer
  (Android 14+).
* **Loader**: interpreter path too long / wrong symlink; guest `init` not
  static; `TYLOADER` unset.
* **Virtual root**: paths not translated → `init` can't find `/system`.
* **Graphics**: socket name mismatch; no fd passing → black frames; no HWC.
* **Binder**: `servicemanager` never comes up → framework hangs.
* **Input**: device node not created / EventHub can't open it.
* **Process limit**: too many child processes → the OS kills the container.

Each project's logs give the exact failure (twoyi: boot-log texture + kmsg;
Virtual Master: `VMHealthMonitor` + `vm_boot_failed`; VMOS Pro: engine state + helper log).
