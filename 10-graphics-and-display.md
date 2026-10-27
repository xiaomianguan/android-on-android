# 10 — Graphics & Display (deep dive)

Graphics is where all four systems converge most closely. The official emulator
defined a **wire protocol** for GLES/EGL/render-control; every container project
reuses it almost verbatim and only swaps the transport. This doc is the
authoritative reference for that subsystem.

---

## 1. The two independent graphics problems

A container must solve **two** graphics problems:

1. **App rendering** — a guest app calls `glDrawArrays`. The call must reach the
   host GPU.
2. **System composition** — the guest `SurfaceFlinger` must composite its layers
   into the framebuffer the user sees (the app's `SurfaceView`).

The emulator's design solves both with one protocol:

```
APP GLES                         SYSTEM COMPOSITION
guest app                        guest SurfaceFlinger
  └─ libEGL/libGLESv2              └─ HIDL composer@2.1
       └─ libEGL_emulation              └─ composer@2.1-impl
            └─ libGLESv2_enc                 └─ hwcomposer.<vendor>.so
                 └──────────┬──────────────────────┘
                            ▼
                  libOpenglSystemCommon.so :: HostConnection
                            │  (transport: pipe or socket)
                            ▼
                  host renderer (libOpenglRender)
                     ├─ renderControl server   (windows/ColorBuffers/layers)
                     ├─ GLESv1/v2 decoders      → translators → host GL
                     └─ FrameBuffer/ColorBuffer/RenderServer/RenderThread
                            ▼
                  host ANativeWindow (SurfaceView) → host GPU
```

---

## 2. The wire protocol (emugen)

From the AOSP `DESIGN` document:

> "Several system libraries inside the emulated guest system … collect the
> sequence of EGL/GLES function calls and translate then into a custom wire
> protocol stream that is sent to the emulator program through a high-speed
> communication channel called a *QEMU Pipe*."

* **`emugen`** generates, from `.types`/`.in`/`.attrib` specs, three kinds of
  source: **encoders** (guest), **decoders** (host), and context-aware wrappers.
* A call becomes an **opcode + marshalled arguments** message. Some calls carry
  a **checksum calculator** (`rcSelectChecksumCalculator`, "GL
  checksumCalculator failure" strings) to detect guest/host divergence.
* **Object handles are integers**: a `ColorBuffer` is an id; a `WindowSurface`
  is an id; a `RenderContext` is an id. The host keeps maps
  (`std::map<uint32_t, SmartPtr<RenderContext>>`, `ColorBufferRef`, …).

### 2.1 renderControl opcodes (reconstructed from exported symbols)

These are the non-GL control ops (guest `lib_renderControl_enc.so` exports, and
the host `libOpenglRender.so` decodes them):

```
Version/EGL:   rcGetRendererVersion, rcGetEGLVersion, rcQueryEGLString,
               rcGetGLString, rcGetNumConfigs, rcGetConfigs, rcChooseConfig
Contexts:      rcCreateContext, rcDestroyContext, rcMakeCurrent
Surfaces:      rcCreateWindowSurface, rcDestroyWindowSurface,
               rcSetWindowColorBuffer, rcFlushWindowColorBuffer
ColorBuffers:  rcCreateColorBuffer, rcOpenColorBuffer, rcOpenColorBuffer2,
               rcCloseColorBuffer, rcUpdateColorBuffer, rcReadColorBuffer,
               rcBindTexture, rcBindRenderbuffer, rcColorBufferCacheFlush
Images/fences: rcCreateClientImage, rcDestroyClientImage,
               rcCreateSyncKHR, rcDestroySyncKHR, rcClientWaitSyncKHR,
               rcDupNativeFenceFDANDROID
Display:       rcGetNumDisplays, rcGetDisplayWidth/Height/DpiX/DpiY,
               rcGetDisplayVsyncPeriod, rcFBSetSwapInterval
Composition:   rcPostLayer, rcPostAllLayersDone          ← HWC path
Misc:          rcGetFBParam, rcSelectChecksumCalculator
```

The host-side public API (`libOpenglRender.so`) is small and stable:

```
initOpenGLRenderer(w,h,dpi,fps)      startOpenGLRenderer(win,w,h,xdpi,ydpi,fps)
setNativeWindow(win)                 resetSubWindow(win,x,y,w,h,fbw,fbh,dpr,zRot)
removeSubWindow(win)                 destroyOpenGLSubwindow()
repaintOpenGLDisplay()               startGBServer()
rcPostLayer(...)  rcPostAllLayersDone()  initRenderControlContext(...)
```

Internally the host renderer is built from:
`RenderServer`, `RenderThread`, `FrameBuffer` (`createColorBuffer`,
`createWindowSurface`, `createRenderContext`, `createClientImage`,
`drainRenderContext`), `ColorBuffer` (`create`, `readPixels`, `readback`,
`reformat`, `post`, `blitFromCurrentReadBuffer`, `bindToTexture`,
`bindToRenderbuffer`), `RenderWindow`, `WindowSurface`, `RenderContext`,
`SocketStream`, `GraphicBuffer`, and the GLES decoders/translators.

---

## 3. Transport: the one thing that changes

The protocol is transport-agnostic. `libOpenglSystemCommon.so` has two stream
implementations: `QemuPipeStream` (goldfish pipe) and `SocketStream`/
`UnixStream` (sockets). The container projects force the socket path.

| System | Transport | Names |
|--------|-----------|-------|
| Official emulator | goldfish pipe | `/dev/goldfish_pipe` / `/dev/qemu_pipe`, pipe name `opengles`/`GLESv2` |
| twoyi | UNIX socket | `/data/data/io.twoyi/rootfs/opengles`, `opengles2`, `opengles3` |
| Virtual Master | UNIX socket | per-VM (`libvm.so`) |
| VMOS Pro | UNIX socket | `%s/rfopengles%d`, `%s/NATIVEBUFFERsocket` |
| threetwi (fork) | none (framebuffer capture) | fake gralloc FIFO + scrcpy |

Evidence: the **same three socket strings** appear in the guest
`libOpenglSystemCommon.so` **and** the host `libOpenglRender.so` for twoyi, so
host and guest rendezvous on those sockets inside the rootfs.

---

## 4. Buffer sharing (gralloc / NATIVE_BUFFER / fences)

To composite, the guest must hand pixel buffers to the host renderer without
copying. The mechanism:

* Guest `gralloc.<board>.so` allocates host-backed buffers.
* The buffer is exported as a **`ColorBuffer`** and/or a **native buffer**
  (`rcCreateClientImage`, `EGL_ANDROID_image_native_buffer`,
  `CreateNATIVE_BUFFEREGLImage`).
* **File descriptors are passed over the socket**:
  `sock_fd_read` / `sock_fd_write` (SCM_RIGHTS) on the host side and
  `HostConnection::recvFd` on the guest side; `rcDupNativeFenceFDANDROID` for
  GPU fences.
* VMOS Pro adds a dedicated `NATIVEBUFFERsocket` for this.

This is the subtlest part of the subsystem: get `ColorBuffer` ids, fd passing
and the `WindowSurface`↔`ANativeWindow` binding right, or you get black frames.

---

## 5. How each project uses it

### 5.1 twoyi

* Host: `libtwoyi.so` (Rust) links `libOpenglRender.so`; `renderer_init` obtains
  the `ANativeWindow` from the Java `SurfaceView`, calls
  `setNativeWindow` + `resetSubWindow` + `startOpenGLRenderer`.
* Guest: `hwcomposer.twoyi.so` (`ro.hardware.hwcomposer=twoyi`) uses
  `HostConnection::get()` + `HostConnection::rcEncoder()` and posts layers via
  `rcPostLayer` / `rcPostAllLayersDone`; apps use `libEGL_emulation` →
  `libGLESv2_enc`.
* Transport: UNIX sockets `rootfs/opengles{,2,3}`.

### 5.2 Virtual Master

* The emulator renderer is inside `libvm.so` (`initOpenGLRenderer`,
  `createOpenGLSubwindow`, `destroyOpenGLSubwindow`, `repaintOpenGLDisplay`,
  `setOpenGLDisplayRotation`, `stopOpenGLRenderer`, `init_gles1_dispatch`).
* Transport: per-VM socket; input separately at `vm/vm%d/dev/input/touch`.

### 5.3 VMOS Pro

* `libvmtools.so` / `libvmkrntools.so` contain the full renderer
  (`RenderServer`, `RenderThread`, `GLESv1/2Decoder`,
  `renderControl_server_context_t`, `ColorBuffer::*`, `SocketStream`).
* JNI: `vmtools_init`, `vmtools_resetup_window(Surface,…)`,
  `vmtools_getFrameBufferRGBA`, `vmtools_getScreenShotBitmap`,
  `vmtools_getFps`, `vmtools_enableFrameBufferCapture`.
* Transport: `%s/rfopengles%d` + `%s/NATIVEBUFFERsocket`.

### 5.4 threetwi / headless variant

Instead of the GL renderer, it creates a **fake gralloc device as a FIFO**
(`dev/graphics/gralloc0`, `dev/shm/gralloc_fb`) and captures the guest
framebuffer, then streams it (scrcpy). This proves the display path is
swappable and is a good fallback for headless/server use.

---

## 6. Reuse plan (clean-room)

1. Build `libEGL_emulation`, `libGLESv1_CM_emulation`, `libGLESv2_emulation`,
   `libGLESv1_enc`, `libGLESv2_enc`, `lib_renderControl_enc`,
   `libOpenglSystemCommon` from `device/generic/goldfish-opengl` (Apache-2.0).
2. Build `libOpenglRender` + translators from
   `external/qemu/android/android-emugl` (Apache-2.0).
3. Implement `HostConnection` with `SocketStream` (the AOSP code already has it)
   and a matching listener on the host renderer.
4. Implement a `hwcomposer.<vendor>.so` that posts layers via `renderControl`.
5. Implement `gralloc.<board>.so` with fd-passing so buffers are zero-copy.
6. Keep a **framebuffer-capture fallback** for headless/no-GPU devices.

This is the single largest reuse opportunity and the reason the container
projects are so small compared to the emulator.
