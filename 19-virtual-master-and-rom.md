# 19 — Virtual Master (`com.clone.android.dual.space`) & its ROM — Deep Dive

**Virtual Master** is the newest and most feature-rich implementation in this
study: app **v3.2.80** (versionCode 3280), `minSdk 22`, **`targetSdk 36`
(Android 16)**. Its ROM is a **hybrid Android 11 image**: an AOSP **GSI** system
side plus a **goldfish/emulator** vendor side, dressed up as a Pixel XL
(`marlin`) with Pixel-4 (`flame`) board props. This doc is the exhaustive
reference for the app *and* its ROM, including the **full decryption of the
ROM** and the guest's container-specific modifications.

Sources: `subjects/VM-v3.2.80-official.apk`,
`subjects/rom_11_0_0_1036_3280.zip`, `workspace/jadx/virtualmaster/` (app),
`workspace/vmaster/fs/` (decrypted guest), `workspace/jadx/vmservice/` (guest
agent), and `workspace/notes/vmaster_guest_deepdive.md` (743-line guest report).

---

## 1. Identity

| | Value |
|---|---|
| App label | **Virtual Master** |
| Package | `com.clone.android.dual.space` |
| Version | 3.2.80 (3280), targetSdk 36, compileSdk 36 |
| Backend | `https://api.virtualmaster.app/account/v1`; developer `vmspacetech@gmail.com` |
| ROM | `rom_default_11.0.0`, `os_version 11`, `rom_version 1036`, `minimum_app_ver 3.2.19` |
| Guest (system) | AOSP **GSI** (`ro.product.system.model=mainline`, `generic`) |
| Guest (vendor) | **goldfish/emulator** tree, `ro.product.board=flame`, `ro.board.platform=msmnile` (Pixel 4) |
| Guest fingerprint | `google/marlin/marlin:11/RD2A.211001.002/6934943:user/release-keys` (2021 base, **rebuilt 2026**) |
| Guest product name | **`master`** (`ro.hardware.*=master`, `Launcher3QuickStepMaster`, `master.xml`) |

> Virtual Master is a **distinct product from VMOS** (a different company),
> despite a similar ROM format. Virtual Master uses the
> **`libkrloader64`/`libkr64`** engine; VMOS Pro uses the **prelinker/ylinker**
> engine. See doc 06.

---

## 2. The ROM format — **fully decrypted**

### 2.1 Package layout

`rom_11_0_0_1036_3280.zip` (plain zip):

```
manifest.json     450 B
rom.img           190,034,947 B   (system tree)
rom1.img          176,237,584 B   (vendor / system_ext / odm / product)
superuser.zip       1,343,568 B   (root overlay)
```

`manifest.json`:

```json
{ "id":"rom_default_11.0.0", "display_name":"Android 11.0.0",
  "rom_version":1036, "os_version":11, "support_a32":false, "support_a64":true,
  "minimum_sdk_int":30, "minimum_app_ver":"3.2.19",
  "rom_uri":["rom.img?e=x&c=v2", "rom1.img?e=x&c=v2"],
  "magisk_uri":"asset:///plugins/magisk.zip?h=true",
  "su_uri":"superuser.zip?h=true", "xposed_uri":"",
  "play_uri":"asset:///plugins/play.zip" }
```

### 2.2 The transform: `e=x` = XOR, `c=v2` = 7z

`rom.img`/`rom1.img` are **XOR-encrypted with the 16-byte repeating key
`%z89aviCM0KkbEs9`**, and the result is a **7-Zip archive** (LZMA + BCJ):

* `XOR(rom.img, "%z89aviCM0KkbEs9")` starts with the 7-Zip signature
  `37 7A BC AF 27 1C 00 03`, and the 7z **StartHeaderCRC validates**
  (`0xf363c0d9` == CRC32 of the next 20 bytes). Same for `rom1.img`.
* `7z l` lists 2479 files (rom.img) / 1440 files (rom1.img) under `fs/…`.

The header is self-revealing: the first ~0x20 bytes are a 7z header XORed with
the key; where the plaintext is zero, the ciphertext **equals the key**, so the
string `%z89FviCM0Kk` appears at offset 0x10 (the key with one byte changed by a
non-zero header byte).

> `e` = encryption (`x` = XOR), `c` = container version (`v2` = 7z/LZMA).

### 2.3 App-side extraction (deobfuscated)

`m5/WWWWoWWWWo.java` chooses the output stream from the URI query parameters
(decoded strings): `c` = `v2`, `e` = `x`, `n` = …:

* plain `FileOutputStream` when a param says so;
* a custom **`w4.WWWWoWWWWo(FileOutputStream, key)`** XOR stream when
  `param("e") == "x"`;
* else `Cipher.getInstance("AES")` + `SecretKeySpec("%z89aviCM0KkbEs9")`
  (`DECRYPT_MODE`) → an AES path (used for `superuser.zip` and the app's report
  file `vminfo.txt`).

7z extraction uses `AndUn7z` (`libun7z.so`) on 64-bit devices. The key is in the
deobfuscated strings at `m5/WWWWoWWWWo.java:90,93` and `n4/WWWW.java:291`.

### 2.4 `superuser.zip` — AES-ECB (solved)

Encrypted with a **different scheme** (`?h=true`): **AES-ECB** with the same key
`%z89aviCM0KkbEs9` (Java `Cipher.getInstance("AES")` = AES/ECB/PKCS5Padding).
Decrypting yields a normal zip:

```
system/app/Superuser/Superuser.apk        (1,070,720 B)
system/app/Superuser/oat/arm64/{Superuser.odex,Superuser.vdex}
system/xbin/daemonsu                      (1,165,256 B)
```

i.e. the **ChainsDD Superuser root overlay**, applied by the app's
`SuperuserTask`.

### 2.5 Decrypted guest size

* `rom.img` → 726 MB / 2479 files (the `system` tree).
* `rom1.img` → 549 MB / 1440 files (`vendor`, `system_ext`, `odm`, `product`).
* Combined ≈ **1.28 GB** of Android 11 userspace.

---

## 3. The app (host side)

### 3.1 Components

| Artifact | Role |
|---|---|
| `libvm.so` (9.7 MB) | The engine (OLLVM-obfuscated). `System.loadLibrary("vm")`; `NativeHelper.startOS(vmid, osVer, romVer, <dataDir>/lib64)`. |
| `libkrloader64.so` | Host loader (INTERP `/system/bin/linker64`). |
| `libkr64.so` | Guest linker = patched bionic `linker64` **+ ShadowHook 2.0.1** (hooks `soinfo::call_constructors`). |
| `libun7z.so` | 7z extraction. |
| `assets/plugins/{magisk,superuser,xposed,play}.zip` | Root/Play overlays. |

The app-side pipeline (fs-prepare `y4.*` tasks, addon `z4.*` tasks, `startOS`,
event bus, binder bridge, HAL services, VPN/netlink, activation, per-VM layout
`<dataDir>/vm/vm<N>/fs`) is the same engine family as Virtual Master (doc 04).

### 3.2 The engine chain

```
VMApp → libvm.so
  └─ startOS(vmid, 11, 1036, <dataDir>/lib64)
       └─ libkrloader64.so            INTERP /system/bin/linker64
            ▼
          libkr64.so                  INTERP <dataDir>/lib64/libkrloader64.so
            │  patched bionic linker + ShadowHook
            ▼
          guest init → zygote64 → system_server → apps
```

See doc 09 §3.

---

## 4. The guest image (decrypted) — a hybrid GSI + goldfish

### 4.1 It is a GSI system + a goldfish vendor, dressed as `marlin`

* **System side** = AOSP **GSI**: `ro.product.system.model=mainline`,
  `ro.product.system.device=generic`, `system/system_ext/build.prop` carries the
  `gsi_system_ext_user.prop` marker, `ro.control_privapp_permissions=disable`.
* **Vendor side** = the **goldfish/emulator** tree: `vendor/ueventd.rc` literally
  says *"These settings are specific to running under the Android emulator"* and
  creates `/dev/goldfish_pipe`, `/dev/goldfish_sync`,
  `/dev/goldfish_address_space`, `/dev/ion`, `/dev/qemu_trace`, `/dev/dri/*`.
  `libqemupipe.master.so` = the renamed goldfish `libqemu_pipe.so`.
* **Masquerade:** the fingerprint says `google/marlin/marlin` (Pixel XL), but
  `ro.product.board=flame`, `ro.board.platform=msmnile`, `ro.product.vendor.model=Pixel 4`,
  and the display is **1080×1920 @320 dpi** — inconsistent with marlin
  (1440×2560 @560). `ro.zygote=zygote64` and empty `abilist32` → **64-bit only**.
* Build dates are **2026** (system 2026-04-29, vendor 2026-05-07) — rebuilt long
  after the 2021 stock base.
* The bare `ro.hardware` property is **absent**; it is supplied at boot from the
  kernel cmdline (`proc/cmdline`: `androidboot.hardware=marlin`), so
  `init.marlin.rc` and `fstab.marlin` are used, while every
  `ro.hardware.<type>=master` selects the Virtual Master HALs.

### 4.2 The container-specific modifications (the whole list)

The guest is stock GSI+goldfish **except** for these:

1. **`master` HALs** — the goldfish/emulator HALs renamed (`ro.hardware.*=master`).
2. **GPU host-call shims** — `gralloc.master.so`/`vulkan.master.so` use custom
   syscalls (§4.3).
3. **A patched binder stack** — `getVMBinderMgr`/`setVMBinderMgr`/
   `initVMBinderDriver` (§4.4).
4. **VM guest services** — `vmserver-service.jar` (a system service) +
   `com.android.vmservice` (a privapp) + `/dev/vm_event` (§4.5).
5. **Platform re-signing** — the whole image is signed with Virtual Master's
   platform key (§4.6).
6. Minor: `Launcher3QuickStepMaster`, a `master` GNSS service,
   `master.xml`/`master-hiddenapi-package-whitelist.xml`, an
   `init.rc` comment (*"allows this to work in containers"*), and
   `vmserver-service.jar` added to `SYSTEMSERVERCLASSPATH` in `init.environ.rc`.

Everything else — `linker64`, `libc.so`, `init`, `libinputflinger`/`libinputreader`
(real evdev), SELinux policy (stock; `/`→`rootfs` is standard AOSP), APEX,
framework — is **stock**.

### 4.3 The GPU HALs are host-call shims (custom syscalls 7001/7004)

`gralloc.master.so` (28 KB) and `vulkan.master.so` (16 KB) are stripped,
control-flow-flattened, and export `entrypoint`/`init`/`init_host_func`
(+ `gralloc_*`/`vulkan_device_open`). Their only real import is **`syscall`**.
Disassembly shows:

```
mov  w0, #0x1b59      ; syscall(7001, "can not find dlopen host call func error")
mov  w0, #0x1b5c      ; syscall(7004)  -> returns a host function table
```

i.e. they fetch a **table of host function pointers** (register/unregister/lock/
unlock for gralloc; Vulkan entry points) via **custom syscall numbers 7001 and
7004** (far outside the Linux aarch64 range → serviced by a **host-side kernel
driver / syscall hook**). This replaces the classic goldfish GLES stack — and
indeed **there is no `vendor/lib64/egl/`, no `libEGL_emulation.so`, no
`libGLESv2_enc.so`, no `lib_renderControl_enc.so`, no `libOpenglSystemCommon.so`**
in this image. This is a clear departure from the emulator's
encoder-over-pipe design and is unique to Virtual Master.

### 4.4 Binder: a patched `VMBinderMgr` (not a separate userspace driver)

There is **no `libvbinder.so`**. Instead the stock libraries are patched:

```
system/lib64/libbinder.so:     android::getVMBinderMgr / setVMBinderMgr / initVMBinderDriver
system/lib64/libhidlbase.so:   android::hardware::getVMBinderMgr / setVMBinderMgr / initVMBinderDriver
system/lib64/libhwbinderd.so:  getVMBinderMgr / setVMBinderMgr / initVMBinderDriver
system/bin/servicemanager:     U android::setVMBinderMgr(...)
system/bin/hwservicemanager:   U android::hardware::setVMBinderMgr(...)
```

`initVMBinderDriver()` maps a small (~1 KiB) host-provided region
(`open`→`mmap`→`close`) and parses a table of (name, value) entries —
**[inferred]** a mapping of host service names to binder handles — then
`setVMBinderMgr(hostMgr, guestMgr)` installs it so guest service lookups can be
redirected to host services. The VNDK copies
(`apex/com.android.vndk.current/lib64/libbinder.so`, `libhidlbase.so`) are
patched the same way. `system/bin/vndservicemanager` is **absent** (its role is
folded into the patched `servicemanager`/`hwservicemanager`). `libbinder.so`'s
string table is **encrypted/obfuscated** (its `.rodata` is only 0x20c bytes and
`strings` yields ciphertext) — the binder device path does not survive in
plaintext.

### 4.5 The VM guest services (`vmserver-service.jar` + `com.android.vmservice`)

* **`system/framework/vmserver-service.jar`** (43 KB, `com.android.server.vmserver.*`:
  `VMServer`, `VMEventManager`, `VMEvents`, `LocalServerSocket`, `IVMEventCallback`)
  is added to `SYSTEMSERVERCLASSPATH` in `init.environ.rc`, so it loads a
  **`SystemService`** into the guest's `system_server`. It connects to
  **`/dev/vm_event`** (FILESYSTEM LocalSocket, UTF + backtick framing) and drives
  app install/uninstall (`PackageInstaller`), key injection
  (`InputManager.injectInputEvent`), clipboard, locale/time, navigation and the
  task stack. See `20-artifacts-plugins-and-control-plane.md` §4 for the full
  command table.
* **`system/priv-app/vmservice/vmservice.apk`** — the privapp
  `com.android.vmservice` ("VM Service", v4.0), granted system permissions via
  `system/etc/permissions/master.xml` (`WRITE_SECURE_SETTINGS`,
  `INSTALL_LOCATION_PROVIDER`, `CONTROL_VPN`, `MANAGE_EXTERNAL_STORAGE`,
  `ACCESS_NOTIFICATIONS`, `READ_CLIPBOARD_IN_BACKGROUND`, …) and hidden-API access
  via `master-hiddenapi-package-whitelist.xml`. Classes:
  `MyApplication` (clipboard monitor), `CmdReceiver` (receives
  `EXECUTE_COMMAND` broadcasts; `vpn.stop`), `notification.NotificationListener`
  (forwards guest notifications as JSON; notification commands),
  `location.LocationServiceV2/V3` + `NetworkLocationProvider` (virtual location),
  `ExportFileActivity`.
* The `com.android.vmcore.action.*` protocol (in `vmserver-service.jar` /
  `vmservice.apk` / `Launcher3QuickStepMaster.apk`): `BOOT_COMPLETED`,
  `VM_SERVER_READY`, `ANDROID_OS_READY`, `START_INSTALL_APP`/`INSTALL_APP_RESULT`,
  `START_UNINSTALL_APP`/`UNINSTALL_APP_RESULT`, `CLIPBOARD_DATA`, `SEND_KEY_EVENT`,
  `EXECUTE_COMMAND`, `OPEN_OUTER_PAGE` (return to the host UI),
  `SYNC_PROP_EVENT`, `SET_GLOBAL_SETTING`, `SET_LOCALE_AND_TIME`,
  `TASK_STACK_CHANGED`, `KILL_APP`, `TOGGLE_RECENTS`,
  `SHOW/HIDE_NAVIGATION_EVENT`, `SET_NAVIGATION_BAR_RTL`,
  `EXPAND/COLLAPSE_NOTIFICATIONS_PANEL`, `SESSION_INSTALL_CALLBACK`.
* `system/system_ext/priv-app/Launcher3QuickStepMaster/` — a modified Launcher3
  with the `OPEN_OUTER_PAGE` host-return action.

### 4.6 Platform re-signing (how it gets platform privilege)

The **entire guest system is re-signed with Virtual Master's own platform key**
(extracted from an APK's `META-INF/CERT.RSA`):

```
Subject/Issuer: C=HK, ST=HK, L=HK, O=virtualmaster, OU=virtualmaster,
                CN=virtualmaster, emailAddress=vmspacetech@gmail.com
RSA-2048, CA:TRUE, valid 2024-03-27 → 2051-08-13
```

Re-signing with a self-issued platform key (plus
`ro.control_privapp_permissions=disable`) is how `com.android.vmservice` (and the
VM framework pieces) obtain platform/system privileges inside the guest.

### 4.7 The container mechanism

The evidence points to a **host-assisted container**, not twoyi's userspace path
virtualization and not a plain mount namespace:

* No bionic path virtualization, no `libvbinder`, no socket input.
* GPU HALs use **custom host syscalls (7001/7004)** → a host kernel hook/driver.
* Binder is patched with `VMBinderMgr` (host↔guest binder bridging).
* `init.rc` carries the comment *"The bind+remount combination allows this to
  work in containers"*, and `/data` etc. are commented out of `fstab.marlin`
  (supplied by the host).
* The activation (wireless ADB / Shizuku / root) provides the privilege to set
  up mounts/device nodes.

So Virtual Master is best described as a **host-assisted, mount-namespace
container with a patched binder layer and a host syscall hook for graphics**,
requiring a privileged activation. This is the most invasive of the three apps.

The exhaustive evidence is in `workspace/notes/vmaster_guest_deepdive.md`.

---

## 5. Similarities and differences

### 5.1 With the official Android emulator (the strongest parallel)

* **It reuses the emulator's goldfish HALs**, renamed `master`: camera
  (`EmulatedCamera_QemuCamera`, `android::QemuClient`), audio (`mic_goldfish`,
  "Emulator host-side ALSA audio emulation"), fingerprint ("cannot write
  fingerprint 'listen' to host", `qemud_channel_open`), vibrator ("Goldfish
  vibrator module"), sensors (`goldfish::MultihalSensors`), all over
  **`/dev/goldfish_pipe`** via `libqemupipe.master.so`.
* **The vendor tree is literally the goldfish tree** (`ueventd.rc`'s own comment
  says so; `/dev/goldfish_*`, `/dev/ion`).
* **Differs**: the emulator runs a real guest **kernel** in QEMU; Virtual Master
  shares the host kernel and virtualizes userspace (with a host syscall hook for
  graphics). And Virtual Master's GPU HALs are **host-call shims**, not the
  emulator's GLES encoders.
* **Differs**: the emulator needs no privilege; Virtual Master re-signs the guest
  with a platform key and needs ADB/Shizuku activation.

### 5.2 With twoyi

* Same family of trick (custom loader, goldfish/emulator HALs, rootfs as a tree),
  but Virtual Master uses a **real-device-style GSI+goldfish image**, the
  `libkrloader64`/`libkr64` (linker + ShadowHook) chain, an in-process
  `libvm.so`, **host-call GPU shims**, a **patched `VMBinderMgr` binder**, a guest
  system-service + privapp agent, and **platform re-signing**.
* twoyi is fully open, rootless, and patches bionic for path translation;
  Virtual Master is obfuscated, needs activation, and does not patch bionic.

### 5.3 With VMOS Pro

* Same ROM format, same backend family, same guest-agent concept. VMOS Pro uses
  the **prelinker/ylinker/krnloader** engine and a `libvmoskernel.so`; Virtual
  Master uses the **krloader/kr** engine and a GSI+goldfish image.
* Both reuse emulator HALs; both have a privileged helper; Virtual Master adds
  platform re-signing and the `VMBinderMgr` patch.

### 5.4 Summary table

| | Emulator | twoyi | Virtual Master | VMOS Pro |
|---|---|---|---|---|
| Guest base | goldfish/ranchu | goldfish 8.1 | **GSI + goldfish (as marlin/flame)** | ranchu/marlin |
| Guest kernel | own (QEMU) | none | none | none (`libvmoskernel`) |
| HALs | goldfish/ranchu | goldfish + custom HWC | **goldfish renamed `master`** | `libhalbox` + ranchu |
| Graphics | pipe → `libOpenglRender` | socket → `libOpenglRender` | **host-call syscalls 7001/7004** | socket → `libvmtools` |
| binder | kernel driver | `libvbinder` userspace driver | **patched `VMBinderMgr`** | sockets + zygotes |
| Bionic | stock | patched (`translate_path`) | stock | stock/shipped linkers |
| Loader | n/a | `libloader.so` | `libkrloader64` + `libkr64`(+ShadowHook) | prelinker + ylinker + krnloader |
| Guest agent | n/a | none | **`vmserver-service.jar` + `com.android.vmservice`** | engine services |
| Signing | AOSP keys | test-keys | **Virtual Master platform key** | — |
| ROM format | system image | `rootfs.7z` | **XOR+7z images, AES overlay** | XOR+7z images |
| Privilege | none | none | wireless ADB/Shizuku/root | wireless ADB/Shizuku/root |

---

## 6. Key constants (Virtual Master)

| Constant | Value |
|---|---|
| ROM XOR key | `%z89aviCM0KkbEs9` (16 bytes) |
| ROM container | 7z (LZMA + BCJ), signature `37 7A BC AF 27 1C` |
| Overlay transform | AES-ECB, same key |
| Guest fingerprint | `google/marlin/marlin:11/RD2A.211001.002/6934943` |
| Board props | `ro.product.board=flame`, `ro.board.platform=msmnile`, 1080×1920@320 |
| HAL prefix | `master` (`ro.hardware.*=master`) |
| GPU host-call syscalls | `syscall(7001)` / `syscall(7004)` |
| Binder patch | `android::{get,set}VMBinderMgr`, `initVMBinderDriver` |
| Guest services | `vmserver-service.jar` (`com.android.server.vmserver`), `com.android.vmservice`, `/dev/vm_event` |
| Command protocol | `com.android.vmcore.action.*` |
| Signing cert | `O/OU/CN=virtualmaster, emailAddress=vmspacetech@gmail.com` (CA:TRUE) |
| Backend | `https://api.virtualmaster.app/account/v1` |
