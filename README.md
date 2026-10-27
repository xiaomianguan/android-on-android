# Android-on-Android Virtualization — Research Base

This `docs/` tree is a clean-room research base for building an independent
implementation of "Android-on-Android" (running a second Android system as a
normal app on a phone). It documents **how the reference implementations work**,
**what they share**, and **how they relate to the official Android emulator
(goldfish / ranchu / AVD)**.

It is intentionally *not* a copy of reverse-engineered code. Where a mechanism
is shown, only the minimum identifying evidence (symbol names, short strings,
one-line snippets) is quoted. All bulk artifacts live under `workspace/`.

> **Read this first:** `01-taxonomy-and-background.md` explains the vocabulary
> and the crucial distinction between *emulation* (the official emulator) and
> *userspace virtualization* (twoyi / Virtual Master / VMOS Pro).

---

## Table of contents

| Doc | Topic |
|-----|-------|
| [01-taxonomy-and-background.md](01-taxonomy-and-background.md) | Virtualization levels, vocabulary, the "container vs. emulator" split |
| [02-official-emulator.md](02-official-emulator.md) | Official Android emulator: QEMU/ranchu/AVD, goldfish devices, the GLES wire protocol, `libOpenglRender` |
| [03-twoyi.md](03-twoyi.md) | twoyi — rootless Android container (open-source app + closed ROM) |
| [04-virtual-master.md](04-virtual-master.md) | Virtual Master (`com.clone.android.dual.space`) — in-process VM with custom linker chain |
| [05-vmos-pro.md](05-vmos-pro.md) | VMOS Pro (`com.vmos.pro`) — encrypted ROMs, ADB/Shizuku helper, media protocol |
| [06-comparison.md](06-comparison.md) | Cross-project mechanism matrix and the "family tree" |
| [07-reimplementation-blueprint.md](07-reimplementation-blueprint.md) | Blueprint for a clean own-code implementation (with licensing notes) |
| [08-evidence-catalog.md](08-evidence-catalog.md) | Where every claim comes from (paths, symbols, strings, artifacts) |
| **[09-native-runtime-and-loaders.md](09-native-runtime-and-loaders.md)** | **Loader chains, guest linkers, ShadowHook, obfuscation, bionic changes** |
| **[10-graphics-and-display.md](10-graphics-and-display.md)** | **The GLES/renderControl wire protocol, renderControl opcodes, buffer/fd sharing, transport** |
| **[11-ipc-protocols-and-binder.md](11-ipc-protocols-and-binder.md)** | **Every host↔guest channel + binder virtualization (3 approaches)** |
| **[12-rom-formats-and-crypto.md](12-rom-formats-and-crypto.md)** | **ROM packaging, `manifest.json`/`rom.ini`, encrypted images, `libcore`, packed inode FS** |
| **[13-hal-input-audio-and-networking.md](13-hal-input-audio-and-networking.md)** | **Virtual HALs, input device server, audio bridge, VPN/netlink** |
| **[14-activation-privilege-and-security.md](14-activation-privilege-and-security.md)** | **Activation flows, permissions, security analysis and recommendations** |
| **[15-app-architecture-and-lifecycle.md](15-app-architecture-and-lifecycle.md)** | **App components, state machines, end-to-end lifecycle** |
| **[16-obfuscation-and-deobfuscation.md](16-obfuscation-and-deobfuscation.md)** | **Protection map, Virtual Master data-div/Java deobfuscation, design guidance** |
| **[17-boot-sequence-end-to-end.md](17-boot-sequence-end-to-end.md)** | **Step-by-step boot narrative for all three + failure points** |
| **[18-building-a-guest-image.md](18-building-a-guest-image.md)** | **Practical guide to turning an AOSP image into a container guest** |
| **[19-virtual-master-and-rom.md](19-virtual-master-and-rom.md)** | **Virtual Master (v3.2.80) + its ROM: format, decryption, guest image, agent** |
| **[20-artifacts-plugins-and-control-plane.md](20-artifacts-plugins-and-control-plane.md)** | **Coverage tracker, decrypted plugin overlays, VMOS Pro assets, the guest `VMServer` control plane, the framework plugin** |
| **[21-libvm-engine.md](21-libvm-engine.md)** | **Virtual Master's `libvm.so` engine — deobfuscated (VPN/TUN + lwIP, netlink, Wi-Fi/HAL forwarders, input, audio, display, binder)** |
| **[22-mystery-register.md](22-mystery-register.md)** | **Closing pass: every resolved mystery, every genuinely-unrecoverable item, and full artifact coverage** |

Raw, unfiltered analysis output is kept separately in
`workspace/notes/` (see the evidence catalog for the index).

---

## The subjects under study

| Subject | Kind | What it is |
|---------|------|------------|
| `subjects/twoyi/` + `subjects/twoyi_0.5.4-*.apk` | Open-source app + binary ROM | twoyi 0.5.4 (Android 8.1 guest), MPL-2.0 app, closed `libloader.so` |
| `subjects/VM-v3.2.80-official.apk` | Closed APK | Virtual Master (`com.clone.android.dual.space`) |
| `subjects/vmospro.apk` | Closed APK | VMOS Pro (`com.vmos.pro`) |
| `subjects/rom_11_0_0_1036_3280.zip` | ROM package | Virtual Master Android 11 ROM (`manifest.json` + encrypted `rom.img`/`rom1.img`) |
| Official Android emulator | AOSP source | `goldfish`/`ranchu` boards, `device/generic/goldfish-opengl`, `external/qemu/android/android-emugl` |

---

## Executive summary (one screen)

1. **The official emulator** is a *full-system* virtualizer. It runs a complete
   Linux kernel + Android inside QEMU (TCG JIT or KVM/HVF), presenting virtual
   hardware ("goldfish"/"ranchu" devices, including a real binder driver). A
   guest that wants GPU acceleration encodes EGL/GLES calls into a wire protocol
   (`emugen`, opcode bases 1024/2048/10000) sent over a fast `goldfish_pipe`, and
   a host-side renderer (`libOpenglRender`) replays them on the host GPU.

2. **twoyi, Virtual Master and VMOS Pro are *not* emulators.** They run a second Android
   *userspace* on the **same kernel as the host**, inside the app sandbox. They
   virtualize the Android framework/HAL/graphics/binder layer, not the CPU or
   kernel. This is why they boot in seconds and use almost no extra RAM, and why
   they all need special tricks to overcome the app sandbox.

3. **All three reuse the official emulator's graphics stack** — the guest
   `libEGL_emulation` / `libGLESv2_enc` encoders and a host `libOpenglRender`
   renderer — but **replace the `goldfish_pipe` transport with a UNIX socket**
   (`.../opengles{,2,3}`, `rfopenglesNNN`, `NATIVEBUFFERsocket`), with
   SCM_RIGHTS fd passing for buffers.

4. **All three need a custom ELF loader/linker** to bootstrap a guest whose
   `bionic` (libc + dynamic linker) is a different version than the host's:
   twoyi `libloader.so`, Virtual Master `libkrloader64.so` + `libkr64.so` (a patched
   bionic linker **+ ShadowHook**), VMOS Pro `libkrnloader.so` +
   `libprelinker64.so` + `libylinker64.so` / `libinitlinker64.so` (with a
   seccomp filter and a userspace syscall re-issuer).

5. **All three virtualize `binder` in userspace** because they have no guest
   kernel binder driver: twoyi ships a full userspace binder *driver*
   (`libvbinder.so`, ioctl-compatible over sockets); Virtual Master injects a binder into
   the guest `ActivityManager`; VMOS Pro uses control sockets + delegated
   zygotes.

6. **Rootless is possible, and twoyi proves it.** twoyi implements a **libc/
   linker-level userspace chroot** (`translate_path` prefixes absolute paths
   with the rootfs; `is_need_translate` passes through `/proc`, most of `/dev`
   and `/sys`), **virtualizes UID/GID** via `/dev/vid/<pid>`, re-execs 64-bit
   ELFs through its loader, and materializes every kernel-ish resource (binder,
   input, property service, graphics, `/dev`) as files/sockets inside the rootfs.

7. **Virtual Master** (`com.clone.android.dual.space`, app label "Virtual
   Master", v3.2.80, targetSdk 36) is the newest and most feature-rich variant.
   Its ROM was **decrypted**: the images are **XOR with `%z89aviCM0KkbEs9` → a
   7z archive**, and the overlay is **AES-ECB**. The guest is a **hybrid**: an
   AOSP **GSI** system side + a **goldfish/emulator** vendor side, dressed up as
   a Pixel XL (`marlin`) with Pixel-4 (`flame`) board props. It reuses the
   emulator's goldfish HALs renamed **`master`** (over `/dev/goldfish_pipe`), its
   GPU HALs are **host-call shims using custom syscalls 7001/7004**, its binder
   is a **patched `VMBinderMgr`** in stock `libbinder`, it adds
   **`vmserver-service.jar` + the `com.android.vmservice` privapp** (`/dev/vm_event`),
   and the whole image is **re-signed with Virtual Master's own platform key**.
   See `19-virtual-master-and-rom.md`.

8. **VMOS Pro / Virtual Master add a privileged activation step**
   (wireless ADB / Shizuku / root). For Virtual Master the reason is now
   known: **Android 14+ kills "phantom" child processes**, so it disables
   `activity_manager max_phantom_processes` via ADB. VMOS Pro goes further with
   a helper service (Netty on **TCP 22681**, unauthenticated) and a
   cloud-phone-grade media protocol (`libmci`: 6-byte `csproto` header,
   FlatBuffers, RSA-2048-OAEP handshake, H.264/AAC).

9. **Their "encryption" is mostly theatre:** VMOS Pro's `libcore` uses
   **XOR-with-the-APK-signature-MD5 + Base64** for API strings; Virtual Master's
   ROM uses a **hard-coded XOR key** recoverable from the header. Don't encrypt
   ROMs with a recoverable key.

10. **The best clean-room foundation is AOSP itself**: the goldfish-opengl guest
    encoders and the emulator's host renderer are Apache-2.0 and designed to be
    reused; write your own loader and control plane. See
    `07-reimplementation-blueprint.md` and `18-building-a-guest-image.md`.
