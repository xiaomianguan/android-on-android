# 08 — Evidence Catalog & Method

Everything in `docs/` is traceable to an artifact in `workspace/` or a public
source. This catalog lists the artifacts, the tools used, and the evidence for
the load-bearing claims.

---

## 1. Method and tooling

| Tool | Use |
|------|-----|
| `unzip`, `7z` | Unpack APKs / `rootfs.7z` / ROM packages. |
| `jadx` 1.x | Decompile Java (`classes*.dex`) → `workspace/jadx/{twoyi,vmos,vmospro}`. |
| `apktool` 3.0.3 | Decode resources/manifest (available; manifest dumped with `aapt2` where noted). |
| LLVM binutils (`llvm-readelf`, `llvm-objdump`, `llvm-nm`, `strings`) | ELF analysis of `.so` files and the guest `init`. |
| Python 3 | Entropy analysis, string extraction from fixed-offset `.rodata`. |
| AOSP Gitiles (`android.googlesource.com`) | Official emulator / goldfish-opengl sources. |
| GitHub API / raw | twoyi forks (`watchings/twoyi.`), `vmos-dev/open-vmos-aosp_5.1`. |

A reusable script is at `workspace/notes/analyze_so.sh`.

---

## 2. Artifact inventory (`workspace/`)

```
workspace/
  apk/twoyi/{lib/arm64-v8a/*.so, assets/{rootfs.7z,rom.ini}, classes.dex}
  apk/virtualmaster/{lib/arm64-v8a/*.so, assets/plugins/*, classes*.dex}
  apk/vmospro/{lib/arm64-v8a/*.so, assets/libprelinker*.so, classes*.dex}
  twoyi_rootfs/rootfs/…            (612 MB extracted guest)
  rom/{manifest.json, rom.img, rom1.img, superuser.zip}
  jadx/{twoyi,vmos,vmospro}/       (decompiled Java)
  notes/*.txt, *.md, analyze_so.sh
```

| Artifact | Size | Notes |
|----------|------|-------|
| `subjects/twoyi_0.5.4-03211927-release.apk` | 200 MB | app + `assets/rootfs.7z` |
| `subjects/VM-v3.2.80-official.apk` | 146 MB | Virtual Master |
| `subjects/vmospro.apk` | 42 MB | VMOS Pro |
| `subjects/rom_11_0_0_1036_3280.zip` | 351 MB | Virtual Master Android 11 ROM |
| `workspace/twoyi_rootfs/rootfs` | 612 MB | extracted Android 8.1 goldfish |

---

## 3. Claim → evidence

### 3.1 Official emulator

| Claim | Evidence |
|-------|----------|
| GLES wire protocol over a QEMU pipe | `android/android-emugl/DESIGN` (quoted in `02`), `.../README`. |
| Guest encoders / host renderer split | `goldfish-opengl/README`; `android-emugl/shared/OpenglCodecCommon`. |
| `libOpenglRender` = host renderer | `apk/twoyi/lib/arm64-v8a/libOpenglRender.so` exports `startGBServer`, `rcPostLayer`, `rcPostAllLayersDone`, `initRenderControlContext`, `GLESv2Decoder`, `ColorBuffer::*`. |
| Guest EGL/GLES emulation libs | `twoyi_rootfs/rootfs/system/vendor/lib64/egl/{libEGL_emulation,libGLESv1_CM_emulation,libGLESv2_emulation}.so`; `.../libGLESv2_enc.so`, `lib_renderControl_enc.so`. |

### 3.2 twoyi

| Claim | Evidence |
|-------|----------|
| App execs guest `init` with `TYLOADER` | `app/rs/src/lib.rs` (`Command::new("./init")…env("TYLOADER", loader_path)`). |
| Guest `init` is static | `llvm-readelf -h` → `Type: EXEC`; no `PT_INTERP`/`NEEDED`. |
| Rootfs/loader hard-coded into binaries | strings `/data/data/io.twoyi/rootfs`, `/data/data/io.twoyi/loader64` in `init`, `libc.so`, `libui.so`, `adbd`, `linker64`, `mdnsd`, `su`, `libEGL_emulation.so`, `libOpenglSystemCommon.so`; the threetwi `rom_patcher.rs` enumerates exactly these + derived paths. |
| **Rootless virtual root = libc/linker path translation** | `libc.so` and `linker64` (not stripped) export `translate_path`, `is_need_translate`, `translate_socket`; `translate_path` prefixes absolute paths with the 19-byte `/data/data/io.twoyi/rootfs` string; `is_need_translate` passes through `/proc`, most of `/dev` (incl. `/dev/binder`, `/dev/qemu_pipe`, `/dev/socket/logd`) and most of `/sys`; `libc.so` calls `translate_path` from 30 sites (`openat`/`stat`/`mkdir`/`execve`/`chdir`/…). |
| **UID/GID virtualization** | `read_xid`/`setuid` in `libc.so` read/write a 28-byte struct at `/dev/vid/<pid>` (translated into the rootfs). |
| **`execve` → loader trampoline** | patched `execve` reads `TYLOADER=` (default `/data/data/io.twoyi/loader64`) and `real_execve`s 64-bit ELFs through it. |
| **Custom EventHub** | `libinputflinger.so` `EventHub::openDeviceLocked` does `socket(AF_UNIX)`+`connect(path)` for `/dev/input/*`. |
| **Inline hooks** | `libc.so` contains `And64InlineHook.cpp` (`_GLOBAL__sub_I_And64InlineHook.cpp`). |
| **Graphics sockets** | `HostConnection` uses `<rootfs>/opengles` (main), `opengles2` (fd), `opengles3` (gb fd); renderControl opcodes 10000–10041. |
| **`su`** | static ChainsDD Superuser (`com.thirdparty.superuser`); **no Magisk/Xposed**. |
| `SYS_CHROOT`/`SYS_PTRACE` are **capability names**, not ptrace use | `init` `.rodata` contains `SYS_MODULE, SYS_RAWIO, SYS_CHROOT, SYS_PTRACE, SYS_PACCT, SYS_ADMIN, SYS_RESOURCE` — a `CAP_*` name table (stock init `capabilities.cpp`). |
| Graphics over UNIX sockets | guest `libOpenglSystemCommon.so` strings `/data/data/io.twoyi/rootfs/opengles{,2,3}`, `UnixStream`, `QemuPipeStream`; host `libOpenglRender.so` has the same socket strings. |
| Custom HWC | `system/vendor/lib64/hw/hwcomposer.twoyi.so` imports `HostConnection::get`/`rcEncoder`, links `lib_renderControl_enc.so`; `ro.hardware.hwcomposer=twoyi`. |
| Virtual input devices | `app/rs/src/input.rs`: `TOUCH_PATH=<rootfs>/dev/input/touch`, `KEY_PATH=<rootfs>/dev/input/key0`, streams `device_info` + `input_event`. |
| Userspace binder | `libbinder.so` NEEDED `libvbinder.so`; `servicemanager` NEEDED `libvbinder.so`; `libvbinder.so` strings `BINDER_WRITE_READ`, `VBINDER_CLIENT_*`, `VHWBINDER`. |
| Abstract control socket | `TwoyiSocketServer`/`TwoyiMessenger` (`SOCK_NAME="TWOYI_SOCK"`, `BOOT_COMPLETED`/`SWITCH_HOST`/`SETTINGS`). |
| ADB install into guest | `Installer.java` (`adb -P 9563 connect localhost:22122`, `install -t -r`). |
| ROM is goldfish Android 8.1 | `init.goldfish.rc`, `qemu-props`, `qemud`, `fstab.goldfish`, `default.prop` fingerprint `...:8.1.0/...`. |
| Root support | `system/xbin/su` (static), `init.superuser.rc`, `superuser.zip`. |

### 3.3 Virtual Master

| Claim | Evidence |
|-------|----------|
| In-process VM | `VMApp` `System.loadLibrary("vm")`; `NativeHelper.startOS(int,int,int,String)`. |
| Loader chain | `libkr64.so` INTERP `/data/user/0/com.clone.android.dual.space/lib64/libkrloader64.so`; `libkrloader64.so` INTERP `/system/bin/linker64`, strings `/dev/__properties__`, `property_service`, `mount`, `libbinder.so`. |
| Modified guest linker | `libkr64.so` strings `__dl___linker_init`, `linker: hook ctors and dtors`, `xdl: open from cache`. |
| Guest board | `y4/WWWoWWWo` references `init.marlin.rc`, `init.ranchu.rc`, `fstab.ranchu`, `*.ranchu.so`. |
| Event socket | `s4/WWWWoWWWWo` (`<vmDir>/dev/event`), `s4/WWWoWWWo` (`readUTF`/`` ` ``), `InterfaceC4467WWWWWWWW` (`com.android.vmcore.action.*`). |
| Binder injection | `BinderService.setupBinder`, `getBinderVersion`; `peekService` reflection. |
| VPN/netlink | `VMVpnService`, `VpnTunService` natives, `NetlinkManager`, `VMNetworkConfig`. |
| Activation | `com.phantom.killer.disabler`, `rikka.shizuku.ShizukuProvider`; manifest permissions (`INSTALL_PACKAGES`, `INTERACT_ACROSS_USERS_FULL`, `MANAGE_EXTERNAL_STORAGE`, `DUMP`). |
| ROM format | `manifest.json` schema matches `workspace/rom/manifest.json`. |

### 3.4 VMOS Pro

| Claim | Evidence |
|-------|----------|
| Prelinker chain | `assets/libprelinker64.so` exports `__linker_init`, `linkerload_elf_binary_execv`, `init_seccomp`; `libylinker64.so`/`libinitlinker64.so` INTERP `/data/data/com.vmos.pro/vmb/libprelinker64.so`. |
| Kernel loader | `libkrnloader.so` strings `libos_loader`, `loader: can't mmap addr[0xB0000000]`, `execve`. |
| Per-VM rootfs | `t88`/`fu9`: `dataDir/osimg/r/ot%02x/…`; `guestOSInfo`. |
| Emulator renderer in-app | `libvmtools.so`/`libvmkrntools.so` symbols `RenderServer`, `RenderThread`, `GLESv1/2Decoder`, `renderControl_server_context_t`, `ColorBuffer::*`; strings `%s/rfopengles%d`, `%s/NATIVEBUFFERsocket`. |
| HAL box | `libhalbox.so` JNI `enableHalDrivers`, `CameraPreview`, `GpsChanged`, `WIFIChanged`, `SensorChanged`, `initGSMData`. |
| ROM crypto | `libcore.so` exports `AES_*`, `MD5*`, `pkcs7_*`, `VpiNativeUtils.{initData,decryptData,encryptData,getFileMD5}`. |
| Encrypted images | entropy ≈ 8.0 bits/byte for `rom.img`/`rom1.img`; `superuser.zip` not a plain zip. |
| ADB/Shizuku helper | `C9396` (`_adb-tls-pairing._tcp`), `ns8`/`AdbPairingClient` (`libadb.so`), `C9189` (`cp libvmos.so /data/local/tmp/vmos_starter`), `f98`/`k98` (Netty `22681`). |
| 100 hw services | `HWServiceFactory$HWS1..100`, `:m1..:m100`; `NativeUtil.startzygote_d*`. |
| ExEngine bus | `ExEngineService` (`osimg/socket/`), `du9`/`lu9` (`ex_plugin`/`ex_engine`), `EngineSession` length-prefixed Parcel + FD passing. |
| Packed FS | `libvmkrnfs.so` (`pack`/`unpack`, `Inode` fields). |

### 3.5 ROM package

| Claim | Evidence |
|-------|----------|
| Virtual Master format | `workspace/rom/manifest.json` (`id`, `os_version:11`, `rom_version:1036`, `rom_uri`, `su_uri`, `play_uri`). |
| Encrypted images | Python entropy = 8.0; head bytes `12 00 84 96 46 6a 69 40 …`. |

---

## 4. Open questions / unresolved

1. ~~Exact rootless virtual-root mechanism in twoyi.~~ **Resolved.** The guest
   `libc.so` and `linker64` (both not stripped) contain a path-translation layer
   — `translate_path` (prefixes absolute paths with `/data/data/io.twoyi/rootfs`),
   `is_need_translate` (passthrough list: `/sys/fs/selinux`,
   `/sys/class/android_usb`, …) and `translate_socket` (UNIX socket addresses).
   `libc.so` calls `translate_path` from ~30 sites (`open`/`openat`/`stat`/…).
   See `03-twoyi.md` §4.1.
2. **Provenance of `libvbinder.so`.** Functionally a userspace binder driver;
   whether it originates from AOSP/goldfish or twoyi is unverified.
3. **VMOS Pro ROM crypto details.** Partially resolved: the exported AES has
   no in-binary caller and no importing library; `libcore`'s actual JNI crypto is
   **XOR with the APK-signing-certificate MD5 + Base64** (API obfuscation). The
   ROM images (entropy 8.0, structured prefix, stream-cipher-like) are keyed
   **outside the host APK** — most likely in the guest `libvmoskernel.so` or
   server-side. Exact mode/IV/key remains unproven.
4. **`libkr32`/`libkr32.11`/`libkr32.12` versioning** — the numeric suffixes
   appear to select guest Android versions (`.11`=Android 11, `.12`=Android 12);
   not confirmed.
5. **Host renderer socket setup.** How exactly the host `libOpenglRender`
   creates/listens on `rootfs/opengles*` (vs. being handed a socket) was not
   fully traced; `startGBServer` + the socket strings strongly indicate it
   listens there.

---

## 5. Raw notes index (`workspace/notes/`)

| File | Contents |
|------|----------|
| `emulator_deepdive.md` | 677-line source-cited official-emulator reference (+ `raw/aosp/` mirror) |
| `twoyi_guest_deepdive.md` | 1212-line twoyi guest analysis (path/uid virtualization, binder, input, graphics) |
| `vmaster_guest_deepdive.md` | Virtual Master guest analysis (decrypted ROM, `master` HALs, guest agent, stock-vs-modified) |
| `virtualmaster_java_deobfuscated.md` | 627-line Virtual Master deobfuscated Java analysis (full pipeline, events, activation) |
| `virtualmaster_strings_decoded.txt` / `_unique.txt` | 10,432 / 4,443 decoded Virtual Master strings |
| `virtualmaster_deobf.py` | the Virtual Master XOR deobfuscator |
| `virtualmaster_java_findings.md` | earlier Virtual Master Java report |
| `vmospro_java_findings.md` | VMOS Pro Java orchestration report (487 lines) |
| `vmospro_native_deepdive.md` | VMOS Pro native/protocol report |
| `deobf_datadiv.py` | Unicorn emulator for Virtual Master `.datadiv_decode*` |
| `libvm_strings.py` | the `libvm.so` per-string XOR deobfuscator → `virtualmaster_libvm_strings.txt` (540 strings) |
| `libvm_jni.py`, `libvm_xref.py`, `libvm_calls.py` | libvm.so JNI-table/xref/call analyzers |
| `libvm_strings2.py` | segmenting libvm.so string decoder |
| `trace_libloader.py`, `trace_libloader2.py` | Unicorn tracers for twoyi libloader.so (behaviour recovered) |
| `virtualmaster_core.txt` | ELF/symbol/string analysis of `libkr64`, `libkrloader64`, `libvm` |
| `vmospro_linkers.txt`, `vmospro_core.txt`, `vmospro_more.txt` | VMOS Pro native analyses |
| `analyze_so.sh` | reusable native-lib analyzer |
| `raw/aosp/` | local AOSP source mirror used by the emulator report |

---

## 6. Reproduce

```bash
# Unpack + decompile
unzip -o subjects/vmospro.apk -d workspace/apk/vmospro 'lib/arm64-v8a/*' 'classes*.dex'
jadx -d workspace/jadx/vmospro --no-res workspace/apk/vmospro/classes*.dex

# Native analysis
workspace/notes/analyze_so.sh workspace/apk/vmospro/lib/arm64-v8a/*.so

# twoyi guest
7z x subjects/twoyi_0.5.4-03211927-release.apk -oworkspace/apk/twoyi assets/rootfs.7z lib/arm64-v8a/*
7z x workspace/apk/twoyi/assets/rootfs.7z -oworkspace/twoyi_rootfs

# ELF fields
llvm-readelf -lW workspace/apk/vmospro/lib/arm64-v8a/libylinker64.so | grep -i interp
```
