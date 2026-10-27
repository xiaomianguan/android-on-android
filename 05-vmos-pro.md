# 05 — VMOS Pro (`com.vmos.pro`)

VMOS Pro is the commercial successor to VMOS and the most elaborate of the
three. It keeps the "guest Android userspace on the host kernel" model but adds
**encrypted ROMs, a privileged ADB/Shizuku helper, a full virtual HAL layer, a
media/control protocol, and a packed filesystem format** — closer to a
"cloud-phone" stack than a simple container.

Sources: `subjects/vmospro.apk`; `workspace/jadx/vmospro/`;
`workspace/notes/vmospro_java_findings.md`;
**`workspace/notes/vmospro_native_deepdive.md`** (899-line native report).

---

## 1. Native components

| Library | Role |
|---------|------|
| `libprelinker32/64.so` (assets) | **First-stage interpreter** = a full bionic linker (`__linker_init`, `linkerload_elf_binary_execv`, `init_seccomp`, `install_sigsys_handler`, `install_sigsegv_handler`). |
| `libylinker32/64.so` | Guest linker ("Y linker"); `call_func` (magic trampoline), `jmp_syscall_handler` (user-space `svc`). |
| `libinitlinker32/64.so` | Guest `initlinker`; INTERP = `.../vmb/libprelinker64.so`. |
| `libkrnloader.so` | 5.6 KB **32-bit loader**: `--file <rom>/libvmoskernel.so` → `dlopen` + `dlsym("main")` + call. |
| `libvmtools.so` / `libvmkrntools.so` | In-app emulator renderer + renderControl/GLES decoders + input + socket helpers. |
| `libnative-lib.so` | Process/zygote/rootfs/HAL orchestration (`com.vmos.core.utils.NativeUtil`). |
| `libhalbox.so` | Virtual HALs (camera/GPS/WiFi/GSM/sensors) + `start_qemupipe`. |
| `libvmkrnfs.so` | Packed-inode filesystem (`VMKrnFilesystem`). |
| `libmci.so` | Cloud media/control SDK ("shouzhiyun"); FlatBuffers + FFmpeg 3.3.2. |
| `libcore.so` | Crypto/integrity (tiny-AES, mbedtls MD5, zlib, minizip, PKCS#7). |
| `libvmos.so` | The `vmos_starter` privileged helper. |
| `vmos_start_sh.so` | Shell script that installs/execs the starter. |

---

## 2. Bootstrap chain

```
app (com.vmos.pro)
  └─ libkrnloader.so --file <vmDir>/libvmoskernel.so
        main(): dlopen(libvmoskernel.so); dlsym("main"); call
                on mmap failure (ret 37/65) → execve(self) to retry
        ▼
     libvmoskernel.so  ("guest kernel")   ──►  guest init
        │  interpreter rewritten to the shipped linker
        ▼
   libinitlinker64.so / libylinker64.so   INTERP = .../vmb/libprelinker64.so
        │  libylinker: call_func = trampoline to 0x1234567890ABCDEF (patched)
        │              jmp_syscall_handler = user-space svc (dispatcher @0x73890, tag 999)
        ▼
   libprelinker64.so   INTERP = /system/bin/linker64
        │  __linker_init: auxv AT_BASE → map the linker at fixed addrs
        │  init_seccomp: SIGSYS handler + PR_SET_NO_NEW_PRIVS + BPF whitelist
        │  install_sigsys_handler: re-issue trapped syscalls (mmap=222, rt_sigaction=134)
        │  linkerload_elf_binary_execv: custom ELF loader
        ▼
   guest bionic userspace
```

`libnative-lib.so` confirms the layout: `%s/osimg/r/ot%s/libvmoskernel.so`,
`%s/osimg/r/ot%s/system/bin/initlinker32|64`, `%s/osimg/r/ot%s/system/bin/sh`,
`ANDROID_SOCKET_*`, `--socket-name=zygote`, `/dev/binder`, `LOOP_MOUNTPOINT`.
See `09-native-runtime-and-loaders.md` §4.

---

## 3. Filesystem layout

```
/data/data/com.vmos.pro/
  osimg/r/ot%02x/                 per-VM root
    rootfs/  system/framework/{,oat/arm64}  guestOSInfo  libvmoskernel.so
    upgrade.rl  ot%02x_config  ot%02x_envinfo  ot%02x_rominfo  ot%02x_stage
  osimg/r/config/<id>
  osimg/socket/                   host↔guest bus (ex_engine/ex_plugin)
  osimg/r/<id>/tmp_rom_unpack_dir/
  vmb/                            holds libprelinker*.so
```

`libvmkrnfs.so` defines a **packed-inode filesystem** (`init/pack/unpack`,
`addFile/addDir/modFile/modDir/rmFile/rmDir`, `getInodeByPath`; `Inode` = 11
LE ints: `mode, owner, group, linkCount, times, dataSize, dataOffset, crc32Hash,
xattrSize`), also handling SELinux contexts (`u:object_r:rootfs:s0`).

---

## 4. Process model (`libnative-lib.so`)

`JNI_OnLoad` registers **54** methods on `com.vmos.core.utils.NativeUtil`
(+ `EngineSession`). Key groups:

* **zygote**: `startzygote(bool)`, `startzygote71_32`, `startzygote_32_64`,
  `startzygote_d(int)`, `startzygote_d(int,dataDir,romRootDir)`,
  `startzygote_d_64_32`, `startzygote_d_71_32`, `zygote_mgr`.
  `startzygote` creates `<dir>/zygote64|zygote32|zygote_mgr` and listens on a
  UNIX socket via `rfbListenOnUnixPort("%s/zygote")` (imported from
  `libvmtools`).
* **fork/exec**: `forkAndExecve` uses `vfork()`, `prctl(PR_SET_CHILD_SUBREAPER)`,
  sets `FD_CLOEXEC` on fds 3..0x400, parses a newline/space cmdline, `execve`.
  `forkAndExecveKrn` is the loop-engine variant (`ANDROID_BOOTLOGO`/
  `LOOP_MOUNTPOINT` env).
* **pipes/mem**: `start_pipe(dir,name)` → `start_qemupipe` (goldfish qemud pipe,
  used as `vknel_pipe`); `start_ashmem_server(path)` = AF_UNIX server that
  `dlopen`s `libcutils` and serves `ashmem_create_region` for the guest.
* **fs**: `setsdr1`, `rpllk` (symlink replacement), `deRootfs`, `symlink`,
  `chmod`, `fixLink`.
* **env/props**: `%s/dev/__properties__`, `/dev/socket/property_service`,
  `ANDROID_SOCKET_%s=%d`, `%s=%s/system/bin/app_process64`.

---

## 5. Display / input (`libvmtools.so` / `libvmkrntools.so`)

* Near-identical; only the JNI class prefix differs
  (`vmtools_*` vs `vmkrntools_*`, 16 methods each).
* JNI: `vmtools_init`, `vmtools_resetup_window(2)`, `vmtools_input_event(64)`,
  `vmtools_input_keyevent(64)`, `vmtools_getFrameBufferRGBA`,
  `vmtools_getScreenShotBitmap`, `vmtools_getSize`, `vmtools_reflushFramebuffer`,
  `vmtools_removeSubWindow`, `vmtools_setprop`, `vmtools_enableFpsStats`,
  `vmtools_enableFrameBufferCapture`, `vmtools_getFps`,
  `vmtools_getLastPostedBufferTime`.
* Renderer internals: `initOpenGLRenderer`, `start_renderer`, `createOpenGLSubwindow`,
  `setNativeWindow`, `reSetupSubWindow`, `RenderServer`, `RenderThread`,
  `renderControl_{server,decoder}_context_t`, `GLESv1Decoder`, `GLESv2Decoder`,
  `ColorBuffer` (+ FBO blit), `vminput::engine_handle_input(64)`.

### 5.1 Socket protocol (exact)

* `%s/rfopengles%d` — the renderControl command channel
  (e.g. `<vmDir>/rfopengles22501`; `22501` from `vmkrntools_init(w,h,22501,103)`).
* `%s/NATIVEBUFFERsocket` — native-buffer/EGLImage/gralloc-handle channel.
* `rfconnect_socket(path)` waits for the server socket (`access()` + sleep),
  sets `SO_LINGER{1,0}`/`SO_REUSEADDR`, `connect`s.
* `rfsendcmd`/`rfrecvcmd` exchange a **fixed 524-byte (0x20c) record**
  `{ int32 cmd; char arg[512]; }`; `rf_send_buffer` streams bulk bytes.
* `rfbListenOnUnixPort`/`rfbAcceptOnUnixPort` are the server side
  (`unlink`+`bind`+`listen(5)`; accept spawns a detached thread).
* **FD passing** via `sendmsg`/`recvmsg` (SCM_RIGHTS) on `NATIVEBUFFERsocket`
  (`CreateNATIVE_BUFFEREGLImage`).
* Framebuffer capture: `vmtools_getFrameBufferRGBA` spins on the render thread,
  then reads back the current `ColorBuffer` (FBO blit).

---

## 6. Virtual HALs (`libhalbox.so`)

Exports: `enableHalDrivers`, `setHalVersion`, `initGSMData`, `GpsChanged`,
`GpsSVInfoList`, `GnsSVInfoList`, `WIFIChanged`, `SensorChanged`,
`CameraPreview/Picture/OnStop`, `start_qemupipe`. GSM AT strings:
`+CGDCONT: %d,"%s","%s","%s",0,0`, `+CLCC: …`, `+COPS: %d,2,%s`,
`+CRSM: 144,0,%s`; WiFi `ssid=%s`, `bssid=%s`; `pipe:qemud:boot-properties`.

Audio: `NativeAudioManagerUtil.startAudioPlayer("<rom>/rfaudio")` streams PCM via
`AudioWriteData`; `recordingWrite` feeds host mic into the guest.

---

## 7. The media/control protocol (`libmci.so`)

Vendor "shouzhiyun"; `SO_FILE_VERSION 3.1.51.1`; static FFmpeg 3.3.2
(H.264 video, AAC audio). JNI classes `com.mci.play.*` (`SWRuntime`,
`SWDataSource` (47 methods), `SWLog`, `SWPlayerSoftImpl`, `SWViewDisplay`).

### 7.1 Wire framing

Every message starts with a **6-byte header** (little-endian):

```
struct _csproto_header_t {
    uint8_t  cat;   // category
    uint8_t  id;    // id within category
    uint32_t len;   // payload length
};
// type = (cat << 8) | id ; payload at offset 6
```

Two payload encodings: **raw** (media/input frames) and **FlatBuffers**
(handshake/control/output/checkapp).

### 7.2 Categories

| cat | purpose | examples |
|-----|---------|----------|
| `0x00xx` | handshake ("Shake") | ShakeOnlineReq `0x0000`, ShakeOnlineRes `0x0001`, ShakeAndPlayReq `0x0005`, ShakeOffer/Answer/Candidate `0x0009..0x000d` |
| `0x01xx` | control | ControlAppReq/Res `0x0104/05`, ControlAVFmt `0x0106/07`, ControlAVTrans `0x0108/09`, ControlVideo `0x010d/0e`, ControlResolution `0x0113`, TransparentMsg `0x0119/1a`, ControlInputRes `0x011b` |
| `0x02xx` | input/sensors | InputTouchReq `0x02C8`, InputLocationReq `0x02C9`, InputAccelerometer `0x02CA`, InputGyro `0x02CC`, InputMagnetometer `0x02CD`, InputGravity `0x02D5`, InputKeyboardReq `0x0264`, InputAudioPack `0x02D3`, InputVideoPack `0x02D4` |
| `0x03xx` | output | OutputString `0x0300`, OutputScreenRes `0x0301`, OutputBrightP `0x0304` |
| `0x04xx` | A/V frames | AvAudio `0x0400`, AvVideo `0x0401` (raw: 6-byte header + 1 frame-type byte + data) |
| `0x05xx` | check app | Checkapp `0x0500` |

Transport: **TCP with optional TLS** (`connection_ssl_connect/handshake/pending`,
`use_ssl` param). Handshake: `resolves → connect → (sslHandshaking) →
shakeHands (ShakeOnlineReq) → onShakeOnlineRes → shakeAndPlayReq → startPlaying`.

### 7.3 Crypto

`handshakeEncrypt` does the crypto **in Java via JNI**: Base64-decode a
hard-coded **RSA-2048 public key**, then `Cipher.getInstance("RSA/NONE/OAEPPadding")`
`ENCRYPT_MODE` → `doFinal(clearBytes)`. So the handshake is an **RSA-OAEP
encryption of a session blob to a fixed server key** (key at `libmci.so` 0x2e2558),
plus optional TLS.

---

## 8. Crypto & integrity (`libcore.so`)

Primitives: tiny-AES (`AES_init_ctx`, `AES_CBC/CTR/ECB`), mbedtls MD5, zlib,
minizip (zip read/write), and a **PKCS#7/APK-signature parser**
(`pkcs7::parse_pkcs7`, `get_from_apk`, `get_MD5`, `check_MD5`, `get_apksignmd5`).

### 8.1 Key derivation — the XOR key is the APK signature MD5

`initData(apkPath)` opens the APK's PKCS#7 block, takes the **16-byte MD5 of the
signing certificate**, and returns it hex-encoded. The same 16 bytes are used by
`xor_data` as a **repeating XOR key**. `encryptData`/`decryptData` are simply
**XOR + Base64** — used for **API-response/string obfuscation**, not ROM
encryption (confirmed by Java: `new String(VpiNativeUtils.decryptData(ctx, msg))`).

`unseal` is a **hidden-API/non-SDK restriction bypass** (walks an array, clears a
dword at `+0x3c`), matching the bundled `org.chickenhook.restrictionbypass`.
`getFileMD5`/`getSignatureMD5` are integrity tokens.

### 8.2 ROM images (honest status)

* `rom.img`/`rom1.img` have entropy 8.0 and a **structured common prefix**
  (first 8 bytes identical; `%z89FviCM0Kk` at 0x10; long identical runs), which
  is characteristic of a **stream cipher (e.g. AES-CTR) over a compressed image**.
* **No internal caller of the exported AES functions exists in `libcore.so`**,
  and no shipped library imports them — so the ROM decryption key is **not in the
  host APK's native libs**. Candidates: the guest kernel `libvmoskernel.so`, or a
  server-side key. `?e=x&c=v2` in `manifest.json` tags the scheme.
* Note the `%z89FviCM0Kk` prefix closely resembles the hard-coded AES key
  `%z89aviCM0KkbEs9` that Virtual Master's report code uses — a strong hint the family
  shares key material, but it is not the ROM key by itself.

---

## 9. Host↔guest bus & privileged helper

* `ExEngineService`: `osimg/socket/{ex_engine,ex_plugin}` (UNIX + abstract +
  inet); `EngineSession` frames `[4-byte BE length][Parcel]` and passes FDs
  (`setFileDescriptorsForSend`). Verbs: install/uninstall APK, query files,
  read package path, start intent, volume mute, transfer, clip, ping.
* Named per-VM forwarders: `start`, `zygote{32,64,_mgr}`, `rfopenglesNNN`,
  `rfserver`, `rftouchNNN`, `rfaudio`, `vknel_pipe`, `dev/socket/{bcs,bis,bhs}`.

### 9.1 `libvmos.so` (`vmos_starter`) — exact flow

```
main():
  parse --apk= / --class=
  if (getuid() != 0 && getuid() != 2000)  → "fatal: run vmos from non root nor adb user" exit(6)
  dup stdout/stderr → /data/local/tmp/vmos_helper_service.log
  kill existing com.vmos.helper_service (scan /proc/%d/cmdline,comm,ns/mnt)
  if no --apk: popen("pm path com.vmos.assistant") → apk = buf+8 ("package:")
  start_server(apk, class):
    daemon(0,0); setenv("CLASSPATH", apk)
    execv("/system/bin/app_process",
          {"-Djava.class.path=<apk>", "--nice-name=com.vmos.helper_service",
           "com.vmos.system_api_service.main.Virtual MasterHelperMain"})
```

`vmos_start_sh.so` is the shell script (`%%%STARTER_PATH%%%` substituted at
activation) that copies/chmods/chowns the starter to
`/data/local/tmp/vmos_starter` and runs it. See `14-activation-privilege-and-security.md`.

### 9.2 Helper control server (Java, TCP 22681)

`k98.java` runs a Netty server on **22681**; Gson requests `{feature,data}`:
`feature=2` start VM, `3` stop VM, `4` run command list, `5` (variant),
`6` `System.exit(0)`, `7` probe. **Unauthenticated** — do not replicate.

---

## 10. Comparison anchors

* Shares with twoyi/Virtual Master: goldfish/ranchu guest, custom linker bootstrap,
  emulator graphics stack with socket transport, virtual HALs, socket IPC.
* Distinct: **encrypted ROMs**, **packed inode FS**, a **media/control protocol**
  (`libmci`, FlatBuffers + RSA-OAEP + FFmpeg), a **prelinker + dual linker +
  kernel loader** chain with seccomp, a **54-method process model** with
  delegated zygotes, and a **privileged ADB helper** (TCP 22681).
