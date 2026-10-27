# 03 — twoyi (rootless Android container)

twoyi is the cleanest reference architecture: open source (app side), rootless,
small enough to fully understand, and its guest patches are **not obfuscated**.
It boots a **goldfish Android 8.1** system as a normal app.

Sources: `subjects/twoyi/` (MPL-2.0 app), `subjects/twoyi_0.5.4-*.apk`,
extracted tree `workspace/twoyi_rootfs/rootfs/`, and the exhaustive guest
analysis in `workspace/notes/twoyi_guest_deepdive.md` (1212 lines).

---

## 1. Split into two halves

| Half | Contents | License |
|------|----------|---------|
| **App** | Java UI + Rust core (`libtwoyi.so`), reused emulator libs (`libOpenglRender.so`, `libadb.so`) | MPL-2.0 |
| **ROM** | `rootfs.7z` → full Android 8.1 goldfish system | closed |

App components:

| Artifact | Role |
|----------|------|
| `Render2Activity` | full-screen `SurfaceView`; `Renderer.init(surface, loaderPath, dpi, fps)` |
| `Renderer` (JNI) | `init/resetWindow/removeWindow/handleTouch/sendKeycode` |
| `libtwoyi.so` (Rust) | starts the renderer, execs the guest `init`, input server, key codes |
| `libOpenglRender.so` | emulator host renderer (patched transport) |
| `libloader.so` | the `loader64` trampoline (closed, MBA-obfuscated) |
| `libadb.so` | ADB client for guest installs |
| `libp7zip.so` + `assets/rootfs.7z` | ROM extraction |

---

## 2. The ROM is a goldfish Android 8.1 image

Identity (from `system/build.prop` / `default.prop`):
`ro.build.fingerprint=Android/twoyi_arm64/twoyi_arm64:8.1.0/OC/root03101833:user/test-keys`,
`ro.product.model=Twoyi`, `ro.build.version.sdk=27`, `ro.build.version.release=8.1.0`.
Container markers: `ro.hardware=goldfish`, **`ro.hardware.hwcomposer=twoyi`**,
`ro.kernel.qemu=1`, `ro.kernel.qemu.gles=1`, `ro.treble.enabled=false`,
`ro.product.cpu.abilist32=` (empty → 64-bit only).

`init.goldfish.rc`, `qemu-props`, `qemud`, `fstab.goldfish`, `ro.hardware=goldfish`
confirm the goldfish base.

---

## 3. Boot flow

```
Render2Activity.surfaceCreated()
  Renderer.init(surface, loaderPath, xdpi, ydpi, fps)     [Java→JNI]
    libtwoyi.so : renderer_init()
      • ANativeWindow_fromSurface(surface)
      • startOpenGLRenderer(win,…) → libOpenglRender (listens on rootfs/opengles{,2,3})
      • Command::new("./init")
          .current_dir("/data/data/io.twoyi/rootfs")
          .env("TYLOADER", <libloader.so>).spawn()

guest init (STATIC):
  ueventd (disabled), hwservicemanager + vndservicemanager,
  configstore → class_start core, zygote64 → system_server, SurfaceFlinger
  → guest sends "BOOT_COMPLETED" over the abstract TWOYI_SOCK
```

`init` is statically linked (`ET_EXEC`, no `PT_INTERP`/`NEEDED`) — the only guest
program the host kernel can start.

---

## 4. The virtual root — a libc/linker-level userspace chroot

The guest `libc.so` and `linker64` are **not stripped** and carry a
path-virtualization layer. This is the heart of the rootless container.

### 4.1 `translate_path(result, dirfd, path, follow)`

* `path[0]=='/'` → copy the literal `/data/data/io.twoyi/rootfs` (16-byte SIMD
  copy), then `join_paths(2, buf, prefix, path)` + `canonicalize(buf, follow,
  result, 576)`.
* relative + `AT_FDCWD` → `real__getcwd` then join.
* relative + real dirfd → build `/proc/<pid>/fd/<dirfd>`, `readlinkat` it, join.

### 4.2 Which libc functions translate (30 callsites)

`faccessat/access`, `fchmodat/chmod/fchmod`, `linkat/link`,
`mkdirat/mkdir`, `mknodat/mknod`, `renameat/rename/renameat2`,
`fstatat64/stat/lstat`, `fchownat/chown/lchown`, `symlinkat/symlink`,
`unlinkat/unlink/rmdir`, `exec_shebang`, **`execve`/`execv*`**,
`setxattr/lsetxattr/getxattr/lgetxattr/listxattr/llistxattr/
removexattr/lremovexattr`, `chdir/fchdir`, `inotify_add_watch`,
`utimensat/utimes/utime`, **`__openat3` (open/openat/open64/creat)**,
`readlinkat/readlink`, `statfs`, and `translate_socket`.

Each calls `is_need_translate(path)` first; only if it returns non-zero does it
call `translate_path`.

### 4.3 `is_need_translate` — the passthrough set

Returns 0 (no translation) for:
* any path already containing `/data/data/io.twoyi/rootfs`;
* **`/proc`** and everything under it;
* **`/dev`** and everything under it, **except** the translate set
  `{/dev/.coldboot_done, /dev/__properties__, /dev/input/*, /dev/__kmsg__,
  /dev/__kmsg2__, /dev/log, /dev/socket (generic and /dev/socket/log*),
  /dev/vbinder, /dev/vndbinder, /dev/hwbinder, /dev/vid/*}`.
  ⇒ `/dev/binder`, `/dev/ashmem`, `/dev/null`, `/dev/qemu_pipe`,
  `/dev/goldfish_pipe`, `/dev/socket/logd` are **passed through to the host**;
* **`/sys`** and everything under it, **except** `/sys/fs/selinux/*` and
  `/sys/class/android_usb/*`.

Everything else (`/system`, `/data`, `/vendor`, `/sdcard`, `/storage`, …) is
translated by prefixing the rootfs.

### 4.4 `translate_socket(sa, len)` — AF_UNIX path rewriting

Called from `bind`, `connect`, `accept4`: it copies `sun_path`, applies
`is_need_translate` + `translate_path`, and rewrites it back. This is how the
guest's `EventHub` connecting to `/dev/input/touch` actually reaches
`<rootfs>/dev/input/touch`, while `/dev/socket/logd` is left alone.

### 4.5 UID/GID virtualization (`/dev/vid/<pid>`)

Not a path feature but part of the same patch family. `read_xid(pid)` and
`setuid/setgid/setreuid/setregid/setresuid/setresgid` read/write a **28-byte
struct at `/dev/vid/<pid>`** (cached in a global); `geteuid` consults the cache.
Because `/dev/vid/*` is in the translate set, these pseudo-files live at
`<rootfs>/dev/vid/<pid>` (created by the host side). **Each guest process thus
has a virtual uid/gid while really running as the host app uid.**

### 4.6 `execve` and the `loader64` trampoline

`execve` is heavily patched:
1. translate the program path;
2. do its own `PATH` search for relative paths;
3. read the first `0x80` bytes; if `\x7fELF\x02` (64-bit ELF), **re-exec through
   the loader**: it reads `TYLOADER=` from envp (or `getenv("TYLOADER")`),
   defaulting to `/data/data/io.twoyi/loader64`, then `real_execve(loader, argv,
   envp)`;
4. scripts → `exec_shebang`, default interpreter `#!/system/bin/sh`.

`/data/data/io.twoyi/loader64` is a symlink to the APK's `libloader.so`.

### 4.7 Inline hooks in libc

`libc.so` contains an inline-hook framework compiled from
`bionic/libc/bionic/And64InlineHook.cpp` (`_GLOBAL__sub_I_And64InlineHook.cpp`
makes an instruction pool RWX). So twoyi can hook libc functions in the guest —
its "Xposed-ish" capability.

---

## 5. `libloader.so` — the bootstrap loader

* 51 KB, `INTERP /system/bin/linker64`, `NEEDED libc.so, libdl.so`; imports
  `dlopen/dlsym/dlclose/dlerror/mmap/mprotect/munmap/open/read/lseek/sprintf/
  __libc_init/_exit/__cxa_atexit`.
* `_start` calls `__libc_init` normally; the body is **MBA-obfuscated** (closed).
* The app installs `<dataDir>/loader64` (and `loader32`) symlinks to it.

Role: map the guest `linker64` (which carries the same path-translation layer)
and hand off. See doc 09.

---

## 6. Graphics

```
Guest apps ──libEGL/libGLESv2──► vendor/lib64/egl/libEGL_emulation.so
   └─► libGLESv2_enc.so ──► libOpenglSystemCommon :: HostConnection
SurfaceFlinger ──► composer@2.1 ──► hwcomposer.twoyi.so ──► HostConnection/rcEncoder
                               │
                               ▼
              three UNIX sockets in the rootfs:
                <rootfs>/opengles   main UnixStream (renderControl + GLES)
                <rootfs>/opengles2  fd channel (recvFd)
                <rootfs>/opengles3  graphics-buffer fd channel (getGbFd/recvGb)
                               ▲
Host libtwoyi.so ──► libOpenglRender.so (socketLocalServer/Accept)
   renderControl server + GLES decoders + ColorBuffer + RenderServer → host GL
```

* `HostConnection::get()` uses `UnixStream::connect(<rootfs>/opengles)`;
  `getGbFd`/`recvGb` use `opengles2`/`opengles3`. The classic `QemuPipeStream`
  (`/dev/qemu_pipe`→`/dev/goldfish_pipe`, pipe name `opengles`) is still compiled
  in but **bypassed**.
* `lib_renderControl_enc.so` implements opcodes **10000–10041**
  (`rcGetRendererVersion=10000` … `rcPostAllLayersDone=10036`,
  `rcCreateSyncKHR=10037`, `rcDupNativeFenceFDANDROID=10040`, `rcFBPost2=10041`).
* `hwcomposer.twoyi.so` (`ro.hardware.hwcomposer=twoyi`) is an Anbox-derived
  goldfish HWC that calls `HostConnection::get()` + `rcEncoder()` and runs a
  vsync thread; loaded via `android.hardware.graphics.composer@2.1-impl.so` →
  `HWC2On1Adapter` → legacy `hwc_composer_device_1`.
* `gralloc.goldfish.so` (`ro.hardware=goldfish`) allocates **ashmem** buffers and
  registers them with the host (`rcOpenColorBuffer/2`), using `libGLESv2_enc`/
  `lib_renderControl_enc`.
* `libEGL_emulation.so` references absolute host paths for the GLES emulation
  libs under `<rootfs>/vendor/lib64/egl/`.

---

## 7. Input — a custom EventHub

The guest's `libinputflinger.so` `EventHub` is patched so every `/dev/input`
entry is a **UNIX socket**:

* `EventHub::scanDirLocked("/dev/input")` → `openDeviceLocked(path)` does
  `socket(AF_UNIX,SOCK_STREAM,0)` + `connect(path)` (not `open()`), then
  `read`s the device descriptor blob and constructs a `Device`.
* The `connect` goes through the patched libc `connect`/`translate_socket`, so
  the path resolves to `<rootfs>/dev/input/<name>`.
* Host side (`libtwoyi.so` Rust, `input.rs`) creates `<rootfs>/dev/input/touch`
  (`vtouch`) and `<rootfs>/dev/input/key0` (`vkey`), writes a `device_info`
  struct (what `EVIOCG*` would return), then streams `input_event` records.
* Host `MotionEvent` → `ABS_MT_*` events; `sendKeycode` → `KEY_BACK`.

---

## 8. Binder — `libvbinder.so`, a userspace binder driver

* `libbinder.so` and `servicemanager` link `libvbinder.so`.
* Exports: `virtual_binder_open`, `virtual_binder_ioctl`, `virtual_binder_mmap`,
  `virtual_binder_driver_main`.
* Protocol: full `BINDER_*` ioctls, `BC_*`/`BR_*` commands, node/ref bookkeeping,
  death notifications, transaction stacks, fd fixup; transport =
  `VBINDER_CLIENT_CONNECT/READ_WRITE/FD_CONNECT/IOCTL_CONNECT`, `VHWBINDER`,
  `recvSeqPacket/sendSeqPacket`, SCM_RIGHTS fd passing.
* Memory: `/dev/ashmem` + `memfd_create` (+ `F_ADD_SEALS`).
* Virtual device nodes `/dev/vbinder`, `/dev/vndbinder`, `/dev/hwbinder`,
  `/dev/vid` are translated into `<rootfs>/dev/…`. **`/dev/binder` is *not*
  translated** (it would hit the host kernel binder).

---

## 9. Property service

`libc.so` hard-codes `property_service_socket =
/data/data/io.twoyi/rootfs/dev/socket/property_service`; `PropertyServiceConnection`
connects there. Since the path already contains the rootfs prefix,
`translate_socket` leaves it alone → the guest talks to a **container-owned
property service**, not the host's. The patched `init` also knows
`/dev/socket/%s`, `/dev/vbinder`, `/dev/vid/%d`.

---

## 10. Control channel, ADB, root

* **Control**: abstract `LocalSocket` `TWOYI_SOCK`; messages `PING`,
  `BOOT_COMPLETED`, `SWITCH_HOST`, `SETTINGS`. `BOOT_COMPLETED` releases the
  boot latch.
* **ADB**: `libadb.so` drives `adb -P 9563 connect localhost:22122` then
  `install -t -r`. Guest `adbd` listens on **22122**
  (`setprop service.adb.tcp.port 22122` when `init.svc.servicemanager` runs);
  `adbd` runs with `--root_seclabel=u:r:su:s0`.
* **Root**: a **statically-linked ChainsDD Superuser `su`**
  (`system/xbin/su`, `com.thirdparty.superuser`), `init.superuser.rc`
  (`su --daemon`), `superuser.zip`. **No Magisk/Xposed** in the twoyi rootfs.
* **File exchange**: `TwoyiDocumentsProvider` exposes the guest `sdcard`.

---

## 11. Boot-script deltas vs stock Android 8.1

* `servicemanager` is **not** started from init (`#start servicemanager`);
  instead `hwservicemanager` + `vndservicemanager` are started, and
  `servicemanager.rc`/`vndservicemanager.rc`/`netd.rc`/`vold.rc` use
  `class svc`.
* `class_start core` is gated on `configstore-hal-1-0=running`; `class_start hal`
  on `hwservicemanager.ready && vndservicemanager running`.
* `ueventd` and `logd` starts are commented out; USB rc files not imported;
  the vendor `init.goldfish.rc` import path is dangling (flattened).
* Zygote `onrestart` lines commented out.
* SELinux: a dedicated `rootfs` type for the tree; `su_exec` for `/system/xbin/su`;
  `vndbinder_device` for `/dev/vndbinder`; vendor policy files are 0 bytes.

---

## 12. ROM management & patching

* `rootfs.7z` unpacked by `P7ZipApi` into `/data/data/io.twoyi/rootfs`;
  `rom.ini` (`author=weishu`, `code`, `version`) drives upgrades; 3rd-party
  `rootfs_3rd.7z` supported.
* ~11 binaries embed the rootfs/loader paths, so relocation requires in-place
  patching (threetwi `rom_patcher.rs`: rootfs ≤ 26 chars, loader ≤ 28).

---

## 13. Clean-room takeaways

* Reuse the emulator's graphics stack (Apache-2.0); twoyi only changed the
  transport (three sockets).
* Do **not** copy `libloader.so` (closed) — write a clean loader from AOSP
  `bionic`.
* The path-translation + materialized-`/dev` + userspace-binder trio is the core
  rootless trick; prefer a **runtime** translator (ptrace or a configurable libc
  patch) over hard-coded paths.
* The uid virtualization (`/dev/vid`) and inline hooks are clever but
  optional; note them as advanced techniques.
