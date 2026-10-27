# 18 — Building a Guest System Image (practical guide)

How to turn an AOSP goldfish/ranchu image into a container-capable guest for
your own Android-on-Android runtime. This is the "make the ROM" counterpart to
`07-reimplementation-blueprint.md`.

---

## 1. Start from a goldfish/ranchu AOSP build

```
lunch emu64a-userdebug        # arm64 ranchu (or emu64x for x86_64)
make -j
```

Outputs: `system.img`, `vendor.img`, `ramdisk.img`, `userdata.img`,
`kernel-ranchu`. Unpack the images (`simg2img` → `ext4`/`erofs` mount or
`debugfs`) into a **directory tree** — that tree is your guest rootfs.

> All three projects keep the guest as a directory tree, never a disk image.

---

## 2. Add the container pieces

### 2.1 Path virtualization (rootless) or namespaces (privileged)

Choose one:

* **Rootless** — patch the guest `bionic` (`libc.so` + `linker64`) with a
  `translate_path`/`is_need_translate`/`translate_socket` layer (see
  `03-twoyi.md` §4). Keep a passthrough set for `/proc`, `/sys`, and the host
  sockets. This is the twoyi model and is fully readable.
* **Privileged** — leave bionic alone and use a real chroot + mount namespace
  from a helper (see `14-activation-privilege-and-security.md`).

If you patch bionic, make the rootfs path **configurable** (env var / config
file) rather than hard-coded, so ROMs stay relocatable.

### 2.2 Custom composer (`hwcomposer.<vendor>.so`)

Implement a HWC (classic `hwc_composer_device_1` or modern HWC3) that posts
layers via the emulator `renderControl` protocol over your transport. Set
`ro.hardware.hwcomposer=<vendor>`. Reference: twoyi's `hwcomposer.twoyi.so`
(`HostConnection::get()` + `rcEncoder()`), or AOSP `system/hwc3`.

### 2.3 gralloc

Keep `gralloc.goldfish.so` (or the modern `hals/gralloc`) and ensure it talks to
your host renderer over the socket, with fd passing. Set
`ro.hardware=goldfish` (or your own) and keep `ro.kernel.qemu.gles=1`.

### 2.4 Userspace binder

Either:
* ship a userspace binder driver (twoyi `libvbinder.so` model) and link
  `libbinder`/`servicemanager` against it; or
* keep the guest's own `servicemanager` and inject a binder from the host
  (Virtual Master model); or
* provide control sockets only (minimum viable).

### 2.5 Virtual input

Patch the guest `EventHub` (`libinputflinger.so`) to treat `/dev/input/*` as
UNIX sockets (twoyi), **or** provide a real virtual input device via a small
kernel-independent path. The twoyi `device_info` + `input_event` protocol is the
reference.

### 2.6 Materialized `/dev` and kernel services

Provide at runtime (created by the host app):
```
<rootfs>/dev/input/{touch,key0}
<rootfs>/dev/socket/property_service
<rootfs>/dev/{vbinder,vndbinder,hwbinder,vid}
<rootfs>/opengles{,2,3}
```
and, where needed, `/dev/__properties__`, `/dev/__kmsg__`, `/dev/kmsg`.

### 2.7 A static bootstrap `init`

Compile the guest `init` **statically** (or ship a tiny static bootstrap) so it
is the first process the host kernel can start. It should:
1. read the rootfs/loader paths (env/config),
2. establish the virtual root (or chroot),
3. start the guest userspace.

### 2.8 HALs

Reuse the goldfish/ranchu HALs; add virtual camera/GPS/Wi-Fi/sensors/audio fed
from the host or synthesized (see `13-hal-input-audio-and-networking.md`).

---

## 3. Prune the boot scripts

Kernel-dependent services will fail in a container. Edit
`init.rc`/`init.<board>.rc` to:
* drop/comment `mount_all`, cgroup/cpuset setup, USB rc imports, `ueventd`
  start (twoyi comments these);
* avoid starting `servicemanager` from init if you use a custom binder path;
* keep `zygote`, `surfaceflinger`, `netd`, `logd`, the HALs, `hwservicemanager`,
  `vndservicemanager`;
* set `ro.setupwizard.mode=DISABLED` for a fast first boot;
* set the adb TCP port property if you want guest adb.

---

## 4. Guest identity / properties

Set a clear identity so the framework behaves:
```
ro.product.model=<Name>
ro.product.brand=Android
ro.build.fingerprint=<vendor>/<product>/<device>:<ver>/...:user/test-keys
ro.hardware=goldfish
ro.hardware.hwcomposer=<vendor>
ro.kernel.qemu=1
ro.kernel.qemu.gles=1
ro.treble.enabled=false        # if pre-Treble
ro.zygote=zygote64             # or zygote64_32
config.disable_bluetooth=true
ro.radio.noril=yes
debug.sf.nobootanimation=1
```
(see twoyi's `system/build.prop` for a working set).

---

## 5. SELinux

The guest's policy assumes its own kernel. Either:
* ship a permissive/relaxed policy for the container, or
* map contexts carefully (twoyi adds a `rootfs` type for the tree, `su_exec` for
  `su`, `vndbinder_device` for `/dev/vndbinder`).
Vendor policy files (`nonplat_*`) can be zeroed if unused.

---

## 6. Package and ship

* Package the tree as a **plain, verifiable archive** (tar+zstd, or 7z like
  twoyi) with a manifest: `id`, `display_name`, `os_version`, `rom_version`,
  `support_a32/a64`, `minimum_sdk_int`, checksums.
* Make the rootfs path a **runtime parameter** — never bake it into binaries.
* Support 3rd-party ROMs (`rootfs_3rd.7z`-style) and upgrades (`rom.ini`-style
  metadata).
* If you encrypt (optional), use a **per-install Keystore key**, not a key
  recoverable from the APK signature.

---

## 7. Root / overlays (optional)

* Root inside the guest: a static `su` + a daemon + an app (twoyi's ChainsDD
  Superuser), or Magisk (Virtual Master) by injecting `/sbin/magisk` and an `init.rc`
  block.
* Xposed: `app_process*_xposed`, `libxposed_art.so`, `XposedBridge.jar`.
* Google Play: `Phonesky`, `PrebuiltGmsCore`, `GoogleServicesFramework`.
* These are *guest* overlays; they do not require host privilege.

---

## 8. Compatibility matrix (what to test)

| Guest ver | 64-bit | 32-bit | notes |
|-----------|--------|--------|-------|
| 5.1 | — | ✓ | VMOS Pro supports; different `libui` shims |
| 7.1 | ✓ | ✓ | Virtual Master has `_32` variant |
| 8.1 | ✓ | ✓ | twoyi's base; well understood |
| 9 | ✓ | — | Virtual Master catalogue |
| 11 | ✓ | — | Virtual Master/VMOS Pro catalogue; the supplied ROM |

Per-version differences that bite: `libui.so`/`libhostlibui.so` shims, the
linker version, `soinfo` layout (for hooks), and HWC API (HWC2 vs HWC3).

---

## 9. Checklist

- [ ] AOSP goldfish/ranchu build, unpacked to a tree.
- [ ] Path virtualization (or namespace) with a configurable rootfs path.
- [ ] Static bootstrap `init`.
- [ ] Custom HWC + gralloc wired to the host renderer over a socket.
- [ ] Userspace binder (or injection).
- [ ] Virtual input device server.
- [ ] Materialized `/dev` + property service.
- [ ] Pruned boot scripts; guest identity props.
- [ ] SELinux policy that permits the container.
- [ ] Packaged as a plain archive with a manifest.
- [ ] Tested across guest versions.
