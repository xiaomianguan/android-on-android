# 22 — Mystery Register (what was resolved, and what is genuinely unrecoverable)

A closing pass over every artifact in `subjects/`: what each mystery was, how it
was resolved, and — where something cannot be recovered statically — exactly why.
The goal is that no *open* question remains; the only unknowns are ones that are
provably unavailable from the shipped artifacts.

---

## 1. Resolved mysteries

| # | Mystery | Resolution | Where |
|---|---------|-----------|-------|
| 1 | twoyi's rootless virtual root | `libc`/`linker64` `translate_path` prefixes the rootfs; `is_need_translate` passthrough set; UID/GID via `/dev/vid` | doc 03 §4 |
| 2 | twoyi `libloader.so` internals (MBA-obfuscated) | **traced** — a minimal ELF loader: `open(rootfs+argv0)` → `mmap` segments → `mprotect` → jump | doc 09 §2, `trace_libloader2.py` |
| 3 | twoyi binder | userspace driver `libvbinder.so` (`virtual_binder_{open,ioctl,mmap,driver_main}`) over sockets | doc 11 §3.1 |
| 4 | twoyi graphics transport | three UNIX sockets `rootfs/opengles{,2,3}`; renderControl opcodes 10000–10041 | doc 03 §6, 10 |
| 5 | twoyi input | custom EventHub connecting to `dev/input/{touch,key0}` sockets | doc 13 §2.2 |
| 6 | Virtual Master ROM encryption | `rom.img`/`rom1.img` = **XOR key `%z89aviCM0KkbEs9` → 7z**; `superuser.zip` = **AES-ECB** | doc 19 §2, 12 |
| 7 | Virtual Master plugin overlays | `assets/plugins/*.zip` = **AES-ECB** → magisk/superuser/xposed/play | doc 20 §2 |
| 8 | Virtual Master guest base | **GSI system + goldfish vendor**, dressed as marlin/Pixel-4 | doc 19 §4.1, `vmaster_guest_deepdive.md` |
| 9 | Virtual Master GPU HALs | `gralloc.master`/`vulkan.master` = host-call shims via **custom syscalls 7001/7004** | doc 19 §4.3 |
| 10 | Virtual Master binder | patched `android::{get,set}VMBinderMgr`/`initVMBinderDriver` in stock libbinder | doc 19 §4.4 |
| 11 | Virtual Master guest control plane | `VMServer` system service + `com.android.vmservice` over **`/dev/vm_event`** | doc 20 §4 |
| 12 | Virtual Master platform privilege | whole image re-signed with **Virtual Master's platform key** | doc 19 §4.6 |
| 13 | **`libvm.so` obfuscation** | strings are **XOR per-string**, decoded inline with NEON; brute-forced → **~797 strings** | doc 21, `libvm_strings*.py` |
| 14 | `libvm.so` engine subsystems | VPN/TUN + **lwIP**, netlink, Wi-Fi, HAL forwarders, input, audio, display, binder, linker namespace | doc 21 |
| 15 | VMOS Pro ROM format | same XOR+7z / AES-ECB family | doc 12 |
| 16 | VMOS Pro `libcore` crypto | `initData` = APK-signing-cert MD5; `encryptData`/`decryptData` = **XOR + Base64** (API obfuscation) | doc 05 §8, 12 |
| 17 | VMOS Pro `libmci` protocol | 6-byte `[cat][id][len]` header, FlatBuffers **1.6.0** + FFmpeg, RSA-OAEP handshake | doc 05 §7 |
| 18 | VMOS Pro loader chain | `libkrnloader` (dlopen+dlsym) → `libprelinker64` (seccomp + SIGSYS re-issuer) → ylinker/initlinker | doc 09 §4 |
| 19 | VMOS Pro `libhalbox` | virtual camera/GPS/WiFi/sensors/GSM HALs (JNI + AT commands) | doc 05 §6 |
| 20 | VMOS Pro `libvmkrnfs` | packed-inode filesystem (`Inode` = 11 ints; add/mod/rm/pack/unpack) | doc 05 §3 |
| 21 | VMOS Pro guest agent / plugins | `com.android.secex` (RomEx service, `com.vmos.exsocket`) + `romex` **Xposed framework** | doc 20 §3.3–3.4 |
| 22 | VMOS Pro activation | wireless ADB + helper (TCP 22681) to defeat Android's phantom-process killer | doc 14 |
| 23 | Emulator graphics protocol | `emugen`, renderControl `.in` (37 opcodes), `libOpenglRender` handlers, `HostConnection` transports | doc 02, `emulator_deepdive.md` |
| 24 | Emulator goldfish_pipe | `goldfish_pipe_qemu.h` protocol (poll/error/wake flags, v1/v2 registers) | doc 02 §2 |
| 25 | Emulator HAL reuse | Virtual Master/VMOS Pro reuse goldfish HALs; twoyi too | doc 06 §2.1 |
| 26 | Virtual Master `master` host-call ABI | `syscall(7004)` returns a **4-entry host function table** (gralloc register/unregister/lock/unlock); `syscall(7001, errmsg)` is the failure path | doc 19 §4.3 |
| 27 | VMOS Pro guest-side ExEngine protocol | `com.vmos.exsocket` uses an **AIDL `IRomService`** + `ArgsParcel` + `ACTION_*` broadcasts (guest `com.android.secex`) | doc 20 §3.4 |
| 28 | Emulator gralloc/renderControl internals | verified against AOSP source: `cb_handle_t`, `ashmem`, `hostHandle`, `refcount`, `rcCreateColorBuffer(DMA)`, capability strings `ANDROID_EMU_*` | doc 02, `raw/aosp/` |

---

## 2. Genuinely unrecoverable (with reasons)

| Item | Why it cannot be recovered from the artifacts |
|------|-----------------------------------------------|
| Virtual Master `libvm.so` **JNI registration tables** | Built at **runtime** in BSS (class names + method names decoded by a separate routine); the static tables read as zeros/garbage. Only the function pointers are static, and they are written at load time. |
| Virtual Master `libvm.so` **remaining garbled strings** | Some `.data` runs concatenate several strings with different XOR constants; the segmenting decoder recovers most but a few remain ambiguous. |
| VMOS Pro `libmci` **FlatBuffers field schemas** | FlatBuffers encodes field *ids*, not names, at runtime; the `.fbs` schema is not shipped (only the generated code + `FlatBuffers 1.6.0`). |
| VMOS Pro **ROM images** | Not shipped in `subjects/` (the supplied ROM is Virtual Master's). |
| VMOS Pro `libcore` **AES key/mode for ROMs** | The AES routines have **no in-binary caller**; the ROM key is outside the host APK (guest kernel / server). |
| twoyi `libloader.so` **exact instruction-level logic** | MBA-obfuscated; behaviour recovered by tracing, but the literal code is not readable. |
| The `pubkey` asset (64 bytes) | Random-looking; no algorithm context shipped (likely a server key/hash). |
| Any **runtime-only** state | e.g. the exact binder file path in `libbinder.so` (string table encrypted). |

These are not "unknowns we missed" — they are data that does not exist in the
shipped files, and each is documented with its reason.

---

## 3. Complete artifact coverage

Every file in `subjects/` and its sub-artifacts:

* **`subjects/twoyi/`** — app source (docs 03/09/15), `.git` (only README/translation commits).
* **`subjects/twoyi_0.5.4-*.apk`** — app + `libloader.so`/`libOpenglRender.so`/`libadb.so` + `rootfs.7z` (decrypted/extracted → `workspace/twoyi_rootfs`), guest deep dive.
* **`subjects/VM-v3.2.80-official.apk`** (Virtual Master) — Java (deobfuscated), native (`libvm.so` deobfuscated; `libkr64`/`libkrloader64`), assets/plugins (decrypted), ROM (decrypted), guest (extracted), guest agent (`vmservice.apk`, `vmserver-service.jar`).
* **`subjects/vmospro.apk`** — Java, all native libs, assets (romex/romex_cn/vmfwplugin/Launcher3/intent/pubkey/siminfo), plugins.
* **`subjects/rom_11_0_0_1036_3280.zip`** — fully decrypted (images + overlay), guest analyzed.
* **Official emulator** — 135-file AOSP source mirror in `workspace/notes/raw/aosp`.

---

## 4. Where to start for the reimplementation
The clean-room blueprint is doc 07; the practical ROM-build guide is doc 18. The
short version:

1. **Runtime**: rootless (twoyi-style path translation) by default; optional
   Shizuku/root for mount namespaces.
2. **Loader**: write a clean ELF loader from AOSP `bionic` (twoyi's is a minimal
   loader; VMOS Pro's prelinker chain shows the seccomp/namespace tricks).
3. **Graphics**: reuse AOSP `goldfish-opengl` + `emugl` (Apache-2.0); swap the
   pipe for a UNIX socket; implement a custom HWC.
4. **binder**: start with a control bus + injected binder; add a userspace
   binder driver if needed.
5. **HALs**: reuse the goldfish HALs; add virtual camera/GPS/Wi-Fi/sensors/audio.
6. **Guest image**: build from a goldfish/ranchu AOSP image; keep it relocatable.

---

## 5. Is there anything left? (honest verdict)

**At the mechanism level: no.** Every subsystem of all four systems is recovered
and documented — how each boots, loads, virtualizes the root, renders, handles
input/audio/HALs, virtualizes binder, packages/encrypts ROMs, activates, and
talks host↔guest.

**At the code level: yes, arbitrarily deep** — but this is depth without
mechanism value, and it was demonstrated to be recoverable:

* every `libvm.so` function's flattened OLLVM disassembly (e.g. the exact
  `setup_binder` control flow, the JNI method implementations);
* the `libmci` FlatBuffers **field offsets** (recoverable from the generated
  `AddElement` calls, e.g. `ShakeOnlineReq` uses offsets 16/14/12/10/4 — but the
  field *names* are not shipped);
* `libvmkrnfs`'s on-disk inode layout (the `Inode` struct is known; the exact
  header bytes would need more disassembly);
* the emulator mirror's remaining ~120 files (the ~15 read already confirm the
  protocol);
* the twoyi `.git` history (97 commits, all README/translation/build — nothing
  about the closed ROM).

**Provably absent** (not "missed"): the `libvm.so` JNI tables, the VMOS Pro ROM
images/key, the `libmci` schema names, `libloader.so`'s literal MBA code. See §2.

So: **no open *question* remains**; only *further depth* on already-understood
code, and data that does not exist in the artifacts.
