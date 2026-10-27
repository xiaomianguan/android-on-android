# 06 — Comprehensive Comparison: the Apps vs. the Emulator

The synthesis of docs 02–05 and 09–19. This is the "what is the same, what is
different, and how does it relate to the real Android emulator" reference.

**Naming note.** The APK `VM-v3.2.80-official.apk` (`com.clone.android.dual.space`)
has the label **"Virtual Master"** — a product **distinct from VMOS** (a different
company; despite the `com.clone` package name and a similar ROM format). So the
three *apps* under study are: **twoyi**, **Virtual Master**, and **VMOS Pro**
(VMOS's pro version). The fourth system is the **official Android emulator**
(AOSP).

---

## 1. The fundamental split

| | Official emulator | twoyi | Virtual Master | VMOS Pro |
|---|---|---|---|---|
| Class | Full-system emulation | Userspace container | Userspace container | Userspace container |
| Guest kernel | Own (in QEMU) | None (host kernel) | None | None (`libvmoskernel.so`) |
| CPU | Emulated (TCG) / KVM | Native | Native | Native |
| Isolation | Hardware | Userspace virtual root | Userspace + helper | Userspace + helper |
| Guest base | goldfish/ranchu board | goldfish 8.1 | **Pixel XL marlin A11** | ranchu/marlin |
| App version | — | 0.5.4 (2022) | **3.2.80, targetSdk 36** | 3.0.7 |
| License | Apache-2.0 | MPL-2.0 app + closed ROM | proprietary | proprietary |

---

## 2. The strongest parallel: the emulator's stack is reused

### 2.1 HALs

* **Virtual Master literally reuses the emulator's goldfish HALs**, renamed
  `master`: `camera.master` (`EmulatedCamera_QemuCamera`, `android::QemuClient`),
  sensors (`goldfish::MultihalSensors`), `audio.primary.master` (`mic_goldfish`,
  "Emulator host-side ALSA audio emulation"), `fingerprint.master`
  ("cannot write fingerprint 'listen' to host", `qemud_channel_open`),
  `vibrator.master` ("Goldfish vibrator module"), all over **`/dev/goldfish_pipe`**
  via `libqemupipe.master.so`. The vendor tree is the goldfish tree
  (`vendor/ueventd.rc`'s own comment says *"specific to running under the Android
  emulator"*; it creates `/dev/goldfish_pipe`, `/dev/goldfish_sync`,
  `/dev/goldfish_address_space`, `/dev/ion`).
* twoyi ships `gralloc.goldfish` and the goldfish HAL set.
* VMOS Pro ships `libhalbox.so` (its own HAL layer) + ranchu HALs.

### 2.2 Graphics

* All three apps speak the **`renderControl` + GLES encoder/decoder** protocol
  and use an `libOpenglRender`-style host renderer; the only change is the
  transport (`goldfish_pipe` → UNIX socket).
* Virtual Master's `gralloc.master`/`vulkan.master` are **thin, obfuscated shims**
  that fetch a host function table via **custom syscalls `syscall(7001)` /
  `syscall(7004)`** ("can not find dlopen host call func error") — the real
  renderer is host-side, serviced by a **host syscall hook**. There is **no**
  guest GLES encoder stack (`libEGL_emulation`/`libGLESv2_enc`/`lib_renderControl_enc`
  are absent). twoyi patches
  `libOpenglSystemCommon` to a socket; VMOS Pro embeds the renderer in
  `libvmtools`.
* Emulator: guest EGL/GLES → QEMU pipe → host `libOpenglRender` → host GL.

### 2.3 Guest agents

The emulator has no guest agent (the host is the emulator). Virtual Master
introduces a **guest-side privapp agent** (`com.android.vmservice`) driven by
injected broadcasts — a container-specific invention.

---

## 3. The shared "container toolkit"

Every app is assembled from the same short list:

| Need | twoyi | Virtual Master | VMOS Pro |
|---|---|---|---|
| **Bootstrap loader** | `libloader.so` (+ static `init`) | `libkrloader64.so` + `libkr64.so` | `libkrnloader.so` + `libvmoskernel.so` |
| **Guest linker** | patched `linker64` | patched bionic `linker64` (`libkr64`) **+ ShadowHook** | `libylinker64`/`libinitlinker64` via `libprelinker64` |
| **Virtual root** | libc/linker `translate_path` (prefix rootfs, rootless) | **host-assisted container** (mount ns + host syscall hook; no bionic patch) | prelinker + `LOOP_MOUNTPOINT` |
| **binder** | `libvbinder.so` (userspace driver) | patched `VMBinderMgr` in stock libbinder | control sockets + delegated zygotes |
| **Input** | evdev-over-socket device server | real evdev (host-fed) | `rftouchNNN` socket |
| **HALs** | goldfish + custom HWC | **goldfish renamed `master`** (over `/dev/goldfish_pipe`) | `libhalbox` + ranchu |
| **Graphics** | socket → `libOpenglRender` | **host-call syscalls 7001/7004** | socket → `libvmtools` |
| **IPC bus** | abstract `TWOYI_SOCK` | `dev/event` (host) + `/dev/vm_event` + `vmserver-service.jar` (guest) | `osimg/socket/*` (Parcel + FD) |
| **ROM** | `rootfs.7z` (plain) | **XOR+7z images, AES overlay** | XOR+7z images |
| **Root/overlay** | in-guest `su` | plugins (magisk/superuser/xposed/play) | plugins + helper |
| **Activation** | none (pure app) | wireless ADB / Shizuku / root | wireless ADB / Shizuku / root |

---

## 4. Where each app lands in the design space

```
 isolation ────────────────────────────────────────────────────────────────►
 emulator      twoyi                 Virtual Master            VMOS Pro
 (real kernel) (rootless container)  (container + helper)      (container + helper + media)
       │              │                     │                        │
   QEMU/KVM      patched bionic        kr-loader + ShadowHook    prelinker + ylinker
   goldfish HALs goldfish HALs         goldfish→master HALs      libhalbox
   pipe→renderer socket→renderer       host-call shims           socket→libvmtools
```

* **twoyi** = minimal, open, rootless.
* **Virtual Master** = newest, real-device image, emulator HALs, guest privapp
  agent, activation.
* **VMOS Pro** = cloud-phone stack (encrypted ROMs, media protocol, 100-service
  zygote delegation, helper on TCP 22681).

---

## 5. Per-subsystem 4-way matrix

### 5.1 Guest system & boot

| | Emulator | twoyi | Virtual Master | VMOS Pro |
|---|---|---|---|---|
| Guest | goldfish/ranchu | goldfish 8.1 | marlin A11 | ranchu/marlin |
| init | guest kernel's | static patched | stock A11 (`bootstrap/linker64`) | guest kernel's |
| Root scripts | stock board rc | heavily edited | largely stock marlin | edited |
| First process | kernel → init | static `init` | loader → guest linker | kernel loader → guest kernel |

### 5.2 Graphics

| | Emulator | twoyi | Virtual Master | VMOS Pro |
|---|---|---|---|---|
| Host renderer | `libOpenglRender` | `libOpenglRender` | in-app (host-call shims) | `libvmtools` |
| Transport | goldfish pipe | `rootfs/opengles{,2,3}` | host-call / socket | `rfopenglesNNN` |
| Guest EGL | `libEGL_emulation` | `libEGL_emulation` (patched) | injected at runtime | shipped |
| gralloc | goldfish | `gralloc.goldfish` | `gralloc.master` (shim) | ranchu + `NATIVEBUFFERsocket` |

### 5.3 binder

| | Emulator | twoyi | Virtual Master | VMOS Pro |
|---|---|---|---|---|
| Model | kernel driver | userspace driver | **patched `VMBinderMgr`** (in stock libbinder) | sockets + zygotes |
| Nodes | `/dev/binder` | `/dev/{v,vnd,hw}binder` → rootfs | binderfs + `/dev/vm_event` | `bcs`/`bis`/`bhs` |

### 5.4 ROM format

| | Emulator | twoyi | Virtual Master / VMOS Pro |
|---|---|---|---|
| Package | system image(s) | `rootfs.7z` | zip + `manifest.json` |
| Images | raw | tree | **XOR+7z** (`e=x&c=v2`) |
| Key | — | — | `%z89aviCM0KkbEs9` (recoverable) |
| Extras | — | `rom.ini` | magisk/superuser/xposed/play zips |

### 5.5 Activation & security

| | Emulator | twoyi | Virtual Master | VMOS Pro |
|---|---|---|---|---|
| Privilege | none | none | ADB/Shizuku | ADB/Shizuku |
| Guest agent | — | — | `com.android.vmservice` | engine/hw services |
| Notable risk | — | — | injected-broadcast agent | unauth TCP 22681 helper |

---

## 6. Similarities (the shared DNA, restated)

1. All apps reuse the **emulator's graphics protocol and (for Virtual Master)
   its HALs**.
2. All need a **custom loader/linker** for the guest bionic.
3. All **virtualize binder** in userspace.
4. All turn **input into a device/socket server**.
5. All keep the **rootfs as a directory tree**.
6. All **virtualize HALs** (camera/GPS/Wi-Fi/sensors/GSM).
7. All expose a **host↔guest control bus**.
8. All are **goldfish/ranchu/marlin** descendants — the emulator's world.

## 7. Differences (the design choices)

1. **Privilege**: twoyi rootless vs. Virtual Master/VMOS Pro activated.
2. **Virtual root**: patched bionic (twoyi) vs. **host-assisted container**
   (Virtual Master: mount namespace + host syscall hook) vs. prelinker
   (VMOS Pro).
3. **Guest edits**: twoyi heavily edits `init.rc` and patches bionic; Virtual
   Master's guest is **stock GSI+goldfish except** the `master` HALs, the
   patched `VMBinderMgr` binder, the VM services, and **platform re-signing**.
4. **Guest image**: goldfish (twoyi) vs. **GSI system + goldfish vendor dressed
   as marlin/flame** (Virtual Master).
5. **HALs**: goldfish + custom HWC (twoyi) vs. goldfish renamed `master`
   (Virtual Master) vs. custom `libhalbox` (VMOS Pro).
6. **Graphics**: GL renderer over a socket (twoyi/VMOS Pro) vs. **host-call
   syscalls 7001/7004** (Virtual Master).
7. **binder**: userspace driver (twoyi) vs. **patched `VMBinderMgr`** (Virtual
   Master) vs. sockets + zygotes (VMOS Pro).
8. **Display**: GL renderer (all) vs. framebuffer capture (twoyi fork).
9. **ROM protection**: plain (twoyi) vs. XOR+7z + AES overlay (Virtual Master)
   vs. same + packed inode FS (VMOS Pro).
10. **Hooking**: none (twoyi) vs. ShadowHook in the guest linker (Virtual
    Master) vs. HAL/app-process layer (VMOS Pro).
11. **Signing**: test-keys (twoyi) vs. **Virtual Master's own platform key**
    (CA:TRUE, 2024–2051).
12. **Obfuscation**: readable (twoyi guest, VMOS Pro native) vs. OLLVM
    (Virtual Master `libvm.so`, `libkrloader64`, the GPU shims, encrypted
    `libbinder` strings).

---

## 8. One-line summaries

* **Official emulator** — a real machine in software; the source of the HALs and
  the graphics protocol everyone reuses.
* **twoyi** — a rootless container built from a patched bionic and socket-backed
  kernel services; the cleanest, most readable reference.
* **Virtual Master** — the newest: a real marlin Android 11 image, emulator HALs
  renamed `master`, an obfuscated `libvm.so` engine with a linker+ShadowHook
  "kernel", a guest privapp agent, and a recoverable-XOR ROM.
* **VMOS Pro** — a cloud-phone-grade stack: encrypted ROMs, a prelinker chain, a
  virtual HAL box, a media protocol, and a privileged helper.
