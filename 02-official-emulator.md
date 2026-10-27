# 02 — The Official Android Emulator (goldfish / ranchu / AVD)

The official emulator is the **reference implementation** for the graphics,
input, host-channel and control-plane design that the container projects copy.
This doc is the condensed, authoritative reference; the exhaustive source-cited
version is in `workspace/notes/emulator_deepdive.md` (with a local AOSP mirror
under `workspace/notes/raw/aosp/`).

### Source branches (they matter)

| Purpose | Repo / branch |
|---|---|
| QEMU fork + `android/` glue + design docs | `platform/external/qemu` @ `emu-master-dev` |
| **Classic emugl host** (emugen, decoders, translators, `libOpenglRender`) | `platform/external/qemu` @ `emu-master-qemu-release` |
| **Classic emugl guest** (encoders, `OpenglSystemCommon`, gralloc, egl) | `device/generic/goldfish-opengl` @ `emu-master-qemu-release` |
| Modern guest graphics (hwc3, gralloc, codecs) | `device/generic/goldfish-opengl` @ `main` |
| Emulator product/board (ranchu), qemud, input | `device/generic/goldfish` @ `main` |
| Goldfish kernel drivers | `kernel/goldfish` @ `android-goldfish-4.14-dev` |
| Modern renderer backend | `platform/hardware/google/gfxstream` @ `main` |

The design docs (`.../android/docs/`) are the best primary source:
`GOLDFISH-VIRTUAL-HARDWARE.TXT`, `ANDROID-QEMU-PIPE.TXT`, `CPU-EMULATION.TXT`,
`GPU-EMULATION.TXT`, `ANDROID-QEMUD*.TXT`, `DISPLAY-STATE.TXT`, etc.

---

## 1. Machine model

* **QEMU fork**: `hw/` is the device model; `android/` is the glue
  (`android-emu`, `android-emugl`, `android-grpc`, `android-net`, `android-ui`,
  `emu/`, `emulator/`).
* **goldfish vs ranchu**: *goldfish* is the virtual-hardware **family**;
  *ranchu* is the current **machine/product** built on it. Products are
  `emu64a` (arm64), `emu64x` (x86_64), `emu64ar`, `emu64xa`; the board name is
  always `goldfish_<arch>`. Guest devices live in `hw/android/goldfish/*.c`;
  drivers in `kernel/goldfish`.
* **CPU modes**: **TCG** (portable JIT; two TB caches + two TLBs per vCPU,
  kernel/user split) plus accelerators **KVM** (Linux), **HVF** (macOS),
  **WHPX** (Windows), **HAXM/AEHD** (Intel). The fast-pipe *parameter block*
  path exists specifically to make KVM/HAXM cheap (one I/O write per op).
* **AVD**: `config.ini`/`hardware-qemu.ini` `hw.*` properties
  (`hw.gpu.enabled`, `hw.gpu.mode`, `hw.lcd.*`, `hw.ramSize`, …) instantiate the
  devices. Kernel 6.6 split into `goldfish_modules/*.ko` (virtio + goldfish).
* **Guest kernel** ships the binder driver (so the emulator needs **no**
  userspace binder) and the goldfish/ranchu drivers.

### 1.1 Virtual devices

| Device | Purpose |
|---|---|
| `goldfish_device_bus` | platform-bus enumeration |
| `goldfish_interrupt_controller` | 32 IRQs |
| `goldfish_timer` / `goldfish_rtc` | high-res timer / wall clock |
| `goldfish_tty` | serial ports (ttyS0 console, ttyS1 legacy qemud) |
| `goldfish_fb` | legacy framebuffer (RGB565, boot-time) |
| `goldfish_audio` | stereo 44.1 kHz out, mono 8 kHz in |
| `goldfish_battery` | battery/AC + IRQ |
| `goldfish_events` | **input triplets** `<type,code,value>` |
| `goldfish_nand` / `goldfish_mmc` | flash / SD |
| `goldfish_pipe` | **fast pipe** (see §2) |
| `goldfish_sync` | host GPU fence → guest sync framework |
| virtio: `blk`, `console`, `net`, `rng`, `gpu`, `input`, `snd`, `wifi`, `vsock` | modern ranchu |

---

## 2. The `goldfish_pipe` / `qemu_pipe` (the transport everyone swaps out)

Guest usage (`qemu_pipe.h`):

```c
fd = open("/dev/qemu_pipe", O_RDWR);        // fallback /dev/goldfish_pipe
write(fd, "pipe:<name>\0", strlen(name)+6);
// then read()/write() like a socket
```

* **Registers** (v1): `COMMAND`, `STATUS`, `CHANNEL(_HIGH)`, `SIZE`,
  `ADDRESS(_HIGH)`, `WAKES`, `PARAMS_ADDR_LOW/HIGH`, `ACCESS_PARAMS`, `VERSION`.
* **Commands**: `PIPE_CMD_OPEN/CLOSE/POLL/WRITE/WAKE_ON_WRITE/READ/WAKE_ON_READ/
  WAKE_ON_DONE_IO/DMA_HOST_MAP/DMA_HOST_UNMAP`.
* **Protocol**: write a buffer (must fit one page; the driver splits larger
  writes) → read `STATUS` (bytes, 0=EOS, <0=error; `AGAIN` → `WAKE_ON_*` + sleep).
  **Parameter blocks** make one read/write a single I/O write (KVM-friendly).
* **v2 pipe**: one command carries *many* discontiguous guest buffers
  (`goldfish_pipe_command`) and one IRQ can signal up to `MAX_SIGNALLED_PIPES`.
* **goldfish_dma**: `ioctl` a contiguous host-visible DMA region and `mmap()` it
  — the zero-copy path for large color buffers.
* **Pipe service names**: `opengles` (the renderer channel), `GLProcessPipe`
  (per-process puid), `refcount` (gralloc buffer lifetime), `QemuMiscPipe`
  (heartbeat/boot-complete), `qemud:<service>`, `boot-properties`, `sensors`,
  `gsm`, `gps`, `tcp:<port>`, `unix:<path>`.
* **qemud framing**: 4-byte length header (4-char lowercase hex if < 64 KiB,
  else big-endian 32-bit with top bit set).

> **This is the exact interface the container projects replace with a UNIX
> socket.** twoyi keeps `QemuPipeStream` but redirects it to
> `rootfs/opengles{,2,3}`; VMOS Pro uses `rfopenglesNNN`.

---

## 3. The graphics wire protocol (emugl)

### 3.1 emugen (the generator)

Three spec files per API: `.in` (RPC signatures), `.types` (type sizes/formats),
`.attrib` (pointer dirs, lengths, flags, `GLOBAL base_opcode`). Invoked as
`emugen -i spec -E enc | -D dec | -W wrap`, it emits encoder/decoder/wrapper
sources. **Opcodes are per-API bases**: GLES1.1 = **1024**, GLES2.0 = **2048**,
renderControl = **10000**.

### 3.2 Wire format

```
struct Packet { unsigned int opcode; unsigned int packet_len; ...params... };
```

* Requests carry `opcode + packet_len + params`; replies are reply-data only.
* `in`/`inout` pointers: `uint32 len; data[len]`; `out`: length only (data in
  reply). `isLarge` = send without an intermediate copy.
* **Checksums**: per-pipe `ChecksumCalculator`. v0 = none; **v1 = 8 bytes**
  (bit-reversed total length + monotonic read/write counters). The host
  advertises `ANDROID_EMU_CHECKSUM_HELPER_v1` in its GL extension string; the
  guest calls `rcSelectChecksumHelper(1,0)` to enable.
* **State tracking**: `GLClientState`, `GLSharedGroup` (host) and the encoders
  (guest) drop no-ops and expand client-side arrays.

### 3.3 renderControl (the control plane)

37 ops, base 10000. Full list:

```
Version/EGL: rcGetRendererVersion, rcGetEGLVersion, rcQueryEGLString,
             rcGetGLString, rcGetNumConfigs, rcGetConfigs, rcChooseConfig,
             rcGetFBParam (FB_WIDTH/HEIGHT/XDPI/YDPI/FPS/MIN/MAX_SWAP)
Contexts:    rcCreateContext, rcDestroyContext, rcMakeCurrent
Surfaces:    rcCreateWindowSurface, rcDestroyWindowSurface,
             rcSetWindowColorBuffer, rcFlushWindowColorBuffer,
             rcFlushWindowColorBufferAsync
ColorBuffers:rcCreateColorBuffer, rcOpenColorBuffer(2), rcCloseColorBuffer,
             rcUpdateColorBuffer(DMA), rcReadColorBuffer, rcBindTexture,
             rcBindRenderbuffer, rcColorBufferCacheFlush
Posting:     rcFBPost, rcFBSetSwapInterval
ClientImage: rcCreateClientImage, rcDestroyClientImage
Sync:        rcCreateSyncKHR, rcDestroySyncKHR, rcClientWaitSyncKHR,
             rcWaitSyncKHR, rcSelectChecksumHelper, rcSetPuid
DMA:         rcCreateColorBufferDMA, rcUpdateColorBufferDMA
```

The host object model: `FrameBuffer`, `ColorBuffer`, `WindowSurface`,
`RenderContext`, `RenderThread`, `PostWorker`, `ReadbackWorker`, `SyncThread`,
`FenceSync`; the public C ABI is `host/include/OpenglRender/render_api.h`
(`initOpenGLRenderer`, `setNativeWindow`, `resetSubWindow`, `startOpenGLRenderer`,
`rcPostLayer`/`rcPostAllLayersDone` in the gfxstream/vkcereal line).

### 3.4 Guest EGL/GLES flow

`libEGL.so` loads `libEGL_emulation.so` via `/system/lib/egl/egl.cfg`
(`0 1 emulation` / `0 0 android`); `init.ranchu.rc` sets `ro.hardware.egl`
(default `emulation`). A GL call:

```
glDrawArrays → libGLESv2_emulation (gl2.cpp)
  → gl2_entry.cpp (generated) → GL2Encoder (gl2_enc.cpp)
  → IOStream buffer from HostConnection::gl2Encoder()
  → commitBuffer → QemuPipeStream::writeFully → write() on qemu_pipe_open("opengles")
  → kernel goldfish_pipe → QEMU hw/misc/goldfish_pipe.c → libOpenglRender ChannelStream
  → RenderThread → gl2_dec → RenderControl.cpp / GLES2 decoder → host GL
```

### 3.5 HostConnection / transport selection

```cpp
enum HostConnectionType { HOST_CONNECTION_TCP, HOST_CONNECTION_QEMU_PIPE, HOST_CONNECTION_VIRTIO_GPU };
```

Classic: one `HostConnection` **per guest GL thread**; `QemuPipeStream` (4 MiB)
+ `qemu_pipe_open("opengles")`, `TcpStream` → `10.0.2.2:22468`, or
`VirtioGpuStream`. After connect: one `clientFlags` int, then lazy encoder
creation (`glEncoder`, `gl2Encoder`, `rcEncoder`) and capability negotiation
(checksum, native sync, DMA, GLES max version, no-error-state, process puid).
Modern gfxstream generalizes to `qemu-pipe` / `address-space` (`asg`) /
`virtio-gpu-pipe` / `virtio-gpu-asg` via `ro.boot.hardware.gltransport`.

### 3.6 ColorBuffer / gralloc / NATIVE_BUFFER / DMA

* `gralloc_alloc` computes an **ashmem** size and calls
  `rcCreateColorBuffer(w,h,glFormat)` (or `rcCreateColorBufferDMA`) → a 32-bit
  **`hostHandle`** stored in `cb_handle_t` (`magic 0xabfabfab`, `fd`, `dmafd`,
  `usage`, dims, `glFormat/glType`, `ashmem*`, `hostHandle`, `goldfish_dma`).
* `register` → `rcOpenColorBuffer2(hostHandle)`; `free` → `rcCloseColorBuffer`.
* SW access → `rcUpdateColorBuffer`/`rcReadColorBuffer`; DMA →
  `bindDmaContext` + `rcUpdateColorBufferDMA`.
* **`NATIVE_BUFFER` sharing**: `eglCreateImageKHR(EGL_NATIVE_BUFFER_ANDROID)`
  validates the handle, refcounts it, wraps it in an `EGLImage_t`;
  `glEGLImageTargetTexture2DOES` binds the host color buffer as a texture.
* **Fences**: `eglCreateSyncKHR(EGL_SYNC_NATIVE_FENCE_ANDROID)` uses
  `goldfish_sync` when available, else `glFinish`.
* **`refcount` pipe**: gralloc writes each `hostHandle` so the host can free
  buffers if the guest dies.
* **fd passing**: classic = ashmem + `goldfish_dma`; gfxstream = `SCM_RIGHTS`
  (`sock_fd_read/write`). A container needs an fd-passing channel in addition to
  the byte stream.

---

## 4. Composition

* **Classic**: `EmuHWC2` (guest `hwc2_device_t`) talks to the host over
  `HostConnection`/renderControl (`rcFBPost`), or the emugl `combined`
  `Toplevel`/`Display`/`ClientComposer` path (EGLImage blit of the framebuffer).
* **Modern**: AIDL **Composer3** (`goldfish-opengl/system/hwc3`) +
  **DRM/KMS over virtio-gpu** + **gfxstream** host renderer. Frame composers:
  `ClientFrameComposer` (SurfaceFlinger composites; `validateDisplay` forces
  `CLIENT`), `GuestFrameComposer` (CPU/libyuv), `HostFrameComposer`,
  `NoOpFrameComposer`. `AlternatingImageStorage` allows image N to be
  overwritten while producing N+1.

The container projects implement a **custom HWC** that speaks renderControl
(twoyi `hwcomposer.twoyi.so`; Virtual Master/VMOS Pro in their renderer libs).

---

## 5. Input

* **Classic `goldfish_events`**: `<type,code,value>` triplets →
  `drivers/input/keyboard/goldfish_events.c` → guest `EventHub`. Touch =
  `EV_ABS ABS_X/Y/Z + EV_KEY BTN_TOUCH + EV_SYN`; multitouch = `ABS_MT_*`;
  trackball = `EV_REL`; lid = `EV_SW`. Capabilities probed via `SET_PAGE`
  (`PAGE_NAME`, `PAGE_EVBITS`, `PAGE_ABSDATA`).
* **Modern `virtio-input`**: device config from
  `device/generic/goldfish/input/*.{kl,kcm,idc}`
  (`virtio_input_multi_touch_1..11.idc`, `virtio_input_rotary.idc`).
* **Console/gRPC**: `EmulatorController.sendKey/sendTouch/sendMouse/
  injectWheel/streamInputEvent` and the skin UI feed the same device.

The containers replace this with an **evdev-over-socket device server** (twoyi)
or a native input event call (Virtual Master/VMOS Pro).

---

## 6. Renderer modes, gRPC, multi-display

* **Modes**: `host` (translate to desktop GL), `swiftshader[_indirect]`
  (software, headless), `angle_indirect` (ANGLE/D3D), `guest` (deprecated),
  `auto`; `-gpu on|off|auto`. Pluggable via `gles_<name>/` + `ANDROID_EGL_LIB`.
* **virtio-gpu**: the modern transport (`VirtioGpu*`, gfxstream command ring,
  blob resources, timelines).
* **Renderer process model**: separable ("the emulator … only acts as a dumb
  proxy"); classic `host/renderer/` headless multi-emulator; modern gfxstream
  `host/`.
* **gRPC control plane** (`android-grpc`, `EmulatorController`): input, display,
  screenshot, sensors, battery, GPS, logcat, VM state, display configurations —
  a language-agnostic model worth copying for a container's control plane.
* **Multi-display**: guest `MultiDisplayProvider` sends `ADD/DEL/QUERY/BIND/
  SET_DISPLAY` over a QEMU pipe and the layer's `hostHandle`; host
  `QAndroidMultiDisplayAgent`; gRPC `setDisplayConfigurations`.

---

## 7. Reuse & licensing (critical for a clean implementation)

* **Apache-2.0**: `android-emugl`/emugl, `goldfish-opengl`, `gfxstream`.
  **Exception**: the bundled Khronos GL headers under
  `goldfish-opengl/system/include/GLES*` are under the Khronos license
  (redistributable, own terms) — keep them separate.
* **Independently buildable, most reusable**:
  * `emugen` (`android-emugl/host/tools/emugen`) — pure C++ codegen.
  * `shared/OpenglCodecCommon` — `ChecksumCalculator`, `GLClientState`,
    `GLSharedGroup`, `SocketStream`/`TcpStream`.
  * Guest: `system/{GLESv1_enc,GLESv2_enc,renderControl_enc,egl,GLESv1,GLESv2,OpenglSystemCommon,gralloc}`.
  * Host: `host/libs/libOpenglRender` + decoders/translators; or gfxstream `host`.
  * Composer: `system/hwc2` (classic) / `system/hwc3` (modern).
* **Build systems**: classic uses the `common.mk` helper layer
  (`emugl-begin-module`, `GOLDFISH_OPENGL_BUILD_FOR_HOST`); modern uses CMake +
  `gfxstream`. The renderer ABI is stable
  (`host/include/OpenglRender/render_api.h`).

### Practical reuse recipe for a container

1. Implement goldfish-pipe semantics (or virtio-gpu + gfxstream) so the guest
   `QemuPipeStream` can connect to `opengles`; **or** swap in a UNIX socket.
2. Vendor `emugen` + the specs; keep the opcode bases and `ChecksumCalculator`.
3. Build the guest EGL/GLES/encoders into the guest image; install `egl.cfg`,
   set `ro.hardware.egl`.
4. Build `libOpenglRender` (or gfxstream) and expose it via the socket +
   `render_api.h`.
5. Build `gralloc` and keep the `hostHandle` + fd-passing contract.
6. Use `EmuHWC2` (classic) or HWC3+DRM (modern) for composition.
7. Add a gRPC-style control plane for input/screenshot/display.
8. Honor the Khronos header exception.

See `07-reimplementation-blueprint.md` and `10-graphics-and-display.md`.
