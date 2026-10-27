# 07 — Reimplementation Blueprint (clean, own-code)

This is the actionable plan for building an independent Android-on-Android
runtime. It assumes the target is a normal app (no root), with an optional
privileged path when the user grants it. The future app lives in `mashi/`; this
doc is the design input for it.

---

## 0. Goals and non-goals

**Goals**

* Boot a second Android userspace from an app, fast, with low RAM.
* Render the guest into a normal `SurfaceView`.
* Route input, audio, and HAL data both ways.
* Keep everything **own-code or Apache-2.0 AOSP**, never copied from
  reverse-engineered binaries.

**Non-goals (initially)**

* CPU/kernel emulation (that is the emulator's job; we are a container).
* Running on a host without ARM64 (out of scope).
* Bypassing Android security to gain host root.

---

## 1. Licensing map (what you may reuse)

| Component | Source | License | Reuse? |
|-----------|--------|---------|--------|
| Guest GLES/EGL encoders | `device/generic/goldfish-opengl` | Apache-2.0 | ✅ yes |
| Host renderer + translators | `external/qemu/android/android-emugl` | Apache-2.0 | ✅ yes |
| `emugen` | `external/qemu/.../emugen` | Apache-2.0 | ✅ yes |
| bionic (libc/linker) | `bionic/` | Apache-2.0/BSD | ✅ yes |
| goldfish HALs / gralloc | `device/generic/goldfish*` | Apache-2.0 | ✅ yes |
| twoyi app | `twoyi/twoyi` | MPL-2.0 | ⚠️ file-level copyleft; prefer ideas over code |
| `libloader.so`, `libkr*`, `libylinker*`, `libvm.so`, `libvmtools.so`, … | closed APKs | proprietary | ❌ no |
| Virtual Master ROMs | proprietary | — | ❌ no |

Rule of thumb: **take AOSP, take twoyi *ideas*, never take closed binaries or
obfuscated code.**

---

## 2. Target architecture

```
┌──────────────────────────── your app (mashi) ────────────────────────────┐
│                                                                          │
│  ContainerController                                                     │
│    ├─ RomManager        extract/verify/place the guest rootfs            │
│    ├─ Runtime           start/stop the guest, health, events             │
│    └─ RootStrategy      Rootless (ptrace/proot) | Namespaced (root/ADB)  │
│                                                                          │
│  Display                                                                 │
│    ├─ HostRenderer      AOSP libOpenglRender (socket transport)          │
│    └─ SurfaceView       ANativeWindow                                    │
│                                                                          │
│  Input                →  InputDeviceServer (evdev over socket)           │
│  Audio                →  AudioBridge (PCM in/out)                        │
│  HalBox               →  virtual camera/gps/wifi/sensors                 │
│  Ipc                  →  LocalSocket bus (typed messages)                │
│                                                                          │
│  ┌──────────────────── guest container (same kernel) ─────────────────┐  │
│  │  init → zygote → system_server → SurfaceFlinger → apps            │  │
│  │  bionic (clean AOSP build, patched for the rootfs)                │  │
│  │  hwcomposer.<vendor>.so → GLES encoder → socket → HostRenderer    │  │
│  │  binder (userspace) • input (socket device) • property (socket)   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Component-by-component plan

### 3.1 Container runtime (`RootStrategy`)

Offer two strategies behind one interface (this mirrors how the ecosystem
actually solves the problem):

1. **Namespaced (preferred when privileges are available).** With root or an
   ADB/Shizuku helper: `unshare`/`mount --bind` + `chroot` + device nodes.
   This is the classic, robust container path (LXC/redroid-like).
2. **Rootless (ptrace).** With no privileges: a **proot-style ptrace supervisor**
   that intercepts path-taking syscalls (`openat`, `stat`, `chdir`, `execve`,
   `mount`, `chroot`, …) and rewrites guest absolute paths to the rootfs. This
   is well-understood and needs no root. (twoyi instead patched bionic; a
   ptrace supervisor is the cleaner, data-driven equivalent and avoids shipping
   modified system libraries.)

**Recommendation:** implement the **ptrace supervisor first** (works everywhere),
and add the namespace path as an accelerator. Keep the rewrite table small:
`/system`, `/vendor`, `/data`, `/dev`, `/proc`, `/sys`, plus the rootfs-relative
special sockets.

### 3.2 Bootstrap loader

Build a small loader from **AOSP `bionic`** that:

* maps the guest's `linker64` and jumps into `__linker_init`, or
* is set as the guest binaries' `PT_INTERP` (the Virtual Master technique), or
* is a static `init` variant (the twoyi technique).

The cleanest is to compile the guest `init` **statically** (as twoyi does) so it
is the first process, then let the guest's own linker take over for everything
else. Keep the loader's job minimal: *load the guest linker, set the link
namespace to the rootfs, transfer control.*

### 3.3 Virtual root

Choose **one** mechanism and stick to it:

* Rootless: ptrace supervisor (3.1) + materialized `/dev` (see 3.5).
* Privileged: bind-mounts + chroot.

Do **not** hard-code absolute paths into system libraries (twoyi's approach);
it makes ROMs non-relocatable and forces in-place binary patching.

### 3.4 Graphics (reuse AOSP wholesale)

* Guest: build `libEGL_emulation`, `libGLESv1_CM_emulation`,
  `libGLESv2_emulation`, `libGLESv2_enc`, `libGLESv1_enc`,
  `lib_renderControl_enc`, `libOpenglSystemCommon` from
  `device/generic/goldfish-opengl`.
* Host: build `libOpenglRender` from `external/qemu/android/android-emugl`.
* **Transport**: implement a `SocketStream`-backed `HostConnection` (the AOSP
  code already has `SocketStream`/`UnixStream`), and a matching listener on the
  host renderer. This is the same minimal change twoyi/Virtual Master made.
* HWC: implement a `hwcomposer.<vendor>.so` that uses `HostConnection` +
  `renderControl` to post layers (copy twoyi's *interface*, not its code; the
  AOSP HWC3 client composer is the reference).
* Fallback path: framebuffer capture + your own transport (for headless mode),
  as threetwi demonstrates.

### 3.5 Virtualized kernel services

| Service | Plan |
|---------|------|
| **binder** | Implement a userspace binder transport (ioctl-compatible shim linked by `libbinder`, moving transactions over a socket, `/dev/ashmem` for shared memory). Clean-room from the **binder UAPI** (`linux/android/binder.h`) and `frameworks/native/libs/binder`. |
| **input** | A device server that presents `/dev/input/*` (via socket or a small `uinput`-like channel) and streams evdev `input_event`s; convert host `MotionEvent`→`ABS_MT_*`. |
| **property service** | A socket-backed property area (`/dev/__properties__`-style) or a minimal `init` property implementation. |
| **ashmem/memfd** | Use `memfd_create` where possible. |
| **log** | Redirect guest `logd` sockets to the host or a file. |

### 3.6 HAL box

Implement fake HALs behind standard HIDL/AIDL interfaces:
camera, GPS/GNSS, Wi-Fi, sensors, battery, vibrator, audio, and (optionally)
GSM. Feed them from the host where possible (`NetlinkManager`-style) or from
synthetic data. AOSP's `goldfish` HALs are a good starting point.

### 3.7 Audio

Bridge guest audio (HAL → socket) to host `AudioTrack`, and host mic
(`AudioRecord`) into the guest. Use a length-prefixed PCM frame protocol; reuse
the AOSP `audio.primary.goldfish` design.

### 3.8 IPC / control

Use a single, typed `LocalSocket` bus between app and guest (`init`-side
helper) with a small, versioned message format. Avoid VMOS Pro's unauthenticated
TCP command server.

### 3.9 ROM management

* Store the guest as an unpacked directory under the app data dir.
* Prefer a **plain, verifiable** archive (e.g. tar/zstd) with a signed manifest;
  if you encrypt, derive the key from your own app signing key **and** bind it to
  a per-install secret.
* Keep the ROM **relocatable** (no hard-coded paths) — the whole point of using
  a runtime path-rewrite instead of patched binaries.

---

## 4. Suggested milestones

| # | Milestone | Exit criterion |
|---|-----------|----------------|
| 0 | Environment | AOSP goldfish-opengl + emugl build for arm64; `emugen` runs. |
| 1 | Static guest `init` | A clean static `init` runs under the app and prints to a log. |
| 2 | ptrace rootless root | Guest `init` sees `/system`, `/vendor` from the rootfs. |
| 3 | Graphics | Guest `SurfaceFlinger` composites into the app `SurfaceView`. |
| 4 | Input | Touch/keyboard reach the guest `EventHub`. |
| 5 | binder | `servicemanager` + one system service work; guest boots to launcher. |
| 6 | Property/log | Guest boot completes without kernel-dependent failures. |
| 7 | HAL box | Camera/GPS/sensors/Wi-Fi report sane values. |
| 8 | Audio | Playback + mic round-trip. |
| 9 | App install | Install/launch a third-party APK in the guest. |
| 10 | Hardening | SELinux/uid mapping, resource limits, multi-profile. |

---

## 5. Risks and open questions

* **Path rewrite correctness** — syscall interception (ptrace) must handle
  `openat2`, `*at` variants, `readlink`, `execve` argv/env, and `cwd`/`root`
  tracking. This is the hardest part; budget for it.
* **SELinux** — the guest's SELinux policy assumes its own kernel; either ship a
  permissive container policy or map contexts carefully.
* **Process limits** — Android 12+ caps app child processes, and Android 14+
  actively kills **"phantom" (child) processes** (`activity_manager
  max_phantom_processes`, `settings_enable_monitor_phantom_procs`). A container
  with `zygote` + `system_server` + apps will exceed this. Virtual Master / VMOS Pro work
  around it with a privileged activation (`device_config put activity_manager
  max_phantom_processes 2147483647`; `settings put global
  settings_enable_monitor_phantom_procs false`) and by spreading guest process
  creation across many `:mN` services. Plan for this from day one: keep the
  process count low, or offer an optional Shizuku-based relief.
* **`/proc`, `/sys`, `/dev`** — some paths must come from the host kernel (real
  `/proc/stat`, `/sys`); a blanket rewrite breaks them. Use an explicit
  passthrough list.
* **binder semantics** — priority inheritance, death recipients, and FD passing
  are subtle; test against real system services.
* **GPU buffer sharing** — `gralloc`/`NATIVE_BUFFER` sharing between guest and
  host renderer needs a correct `ColorBuffer`/fd path.

---

## 6. What to explicitly avoid (anti-patterns observed)

* Hard-coding rootfs paths into system libraries (twoyi) — kills relocatability.
* Obfuscating the core loader (twoyi/Virtual Master) — makes maintenance impossible.
* Unauthenticated local command servers (VMOS Pro TCP 22681) — severe risk.
* Relying on host root/ADB for core functionality (Virtual Master / VMOS Pro) — limits the
  user base; make it optional.
* Encrypting ROMs with a key recoverable from the APK signature — security
  theatre; use a real per-install secret if you encrypt at all.

---

## 7. First concrete step

Build a **"hello container"**:

1. Extract a minimal rootfs (busybox-static + a static `init`) into the app data
   dir.
2. Implement the ptrace path-rewrite supervisor; run `init` under it.
3. Prove `/system/bin/sh -c 'echo hi > /data/x'` works and writes into the
   rootfs.

Everything else (Android, graphics, binder) is layered on that foundation. The
`mashi/` app should start here.
