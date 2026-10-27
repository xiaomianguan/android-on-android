# 01 — Taxonomy and Background

## 1. What "Android-on-Android" can mean

Running one Android system inside another can be done at very different layers.
The layer determines *everything* about performance, isolation, capabilities,
and the amount of code you must write.

```
 high isolation, high cost                                              ┌──────────────┐
        ▲                                                                │ Full-system  │  Official Android emulator
        │                                                                │ emulation    │  (QEMU + KVM/TCG, guest kernel)
        │                                                                └──────────────┘
        │                                            ┌──────────────┐
        │                                            │ Hardware     │  Rare on phones (needs hypervisor)
        │                                            │ virtualization│
        │                                            └──────────────┘
        │                          ┌───────────────────────────────┐
        │                          │ OS/container virtualization    │  twoyi, Virtual Master, VMOS Pro, LXC, redroid,
        │                          │ (share host kernel)            │  Waydroid, Anbox, "Android in a chroot"
        │                          └───────────────────────────────┘
        │        ┌───────────────────────────────────────┐
        │        │ App/framework virtualization           │  VirtualApp, VirtualXposed, "dual apps",
        │        │ (no second Android system at all)      │  app cloning / multi-account
        │        └───────────────────────────────────────┘
 low isolation, low cost
```

The subjects of this research all sit in the **OS/container virtualization**
band, *except* the official emulator which is full-system emulation.

### 1.1 Full-system emulation (official emulator)

* Emulates (or virtualizes) a **CPU** and a **machine**.
* The guest runs its **own Linux kernel** and its own `init`, `zygote`,
  `SurfaceFlinger`, etc.
* Isolation is at the hardware boundary.
* Cost: hundreds of MB of RAM, a full boot, and (without KVM) slow CPU.

### 1.2 OS / container virtualization (twoyi, Virtual Master, VMOS Pro)

* The guest **shares the host Linux kernel**. There is no guest kernel.
* The guest is a *second Android userspace*: its own `init`, `zygote`,
  `servicemanager`, `SurfaceFlinger`, HALs, package manager, `/data`, etc.
* Isolation is enforced in userspace (namespaces/chroot if privileged, or a
  userspace "virtual root" if not).
* Cost: low RAM, ~1–3 s boot once the ROM is extracted.

This is exactly the difference between *QEMU* and *Docker/LXC* — and, on a
phone, the hard part is that a **normal app has almost no privileges**, so the
container runtime must be built from what an app *can* do.

### 1.3 App/framework virtualization (for contrast)

VirtualApp-style "app cloning" does **not** boot a second Android. It loads a
foreign APK into the *host* process/zygote and fakes `Context`, `ActivityManager`
results, etc. No second `SurfaceFlinger`, no second `system_server`. Out of
scope here, but useful to distinguish: twoyi/Virtual Master boot a *real* Android.

---

## 2. Why it is hard: the app sandbox vs. a real container

A Linux container runtime normally uses kernel facilities that a normal Android
app cannot use:

| Facility | Needs | App can? |
|----------|-------|----------|
| `chroot(2)` | `CAP_SYS_CHROOT` | No |
| `mount(2)` / bind mounts | `CAP_SYS_ADMIN` | No |
| `unshare(CLONE_NEWNS/NEWUSER)` | `CAP_SYS_ADMIN` (user ns limited on Android) | No |
| mknod device nodes | `CAP_MKNOD` | No |
| Access `/dev/binder` | SELinux + group | No |
| Spawn a `setuid` process | `CAP_SETUID` | No |
| Read `/proc/<other>` | ptrace/`hidepid` | Limited |

So a rootless Android container must obtain the *effect* of these facilities by
other means. The reference implementations each choose a strategy:

* **twoyi** — patch the guest's own `bionic`/`init` so absolute guest paths and
  library lookups resolve inside a directory tree (`/data/data/io.twoyi/rootfs`)
  that the app owns. Everything the guest would get from the kernel (binder,
  input, graphics, property service, `/dev` nodes) is **re-materialized as
  files and UNIX sockets inside that tree**. No namespaces, no root.
* **Virtual Master / VMOS Pro** — historically the same idea, but modern versions add a
  **privileged helper** started through the user's own wireless-debugging ADB
  (or `su`/Shizuku) so they can do real `mount`/`chroot`/device work.

---

## 3. The common building blocks

Every one of the three container implementations is assembled from the same
short list of subsystems. This is the mental model to carry through the docs:

```
┌────────────────────────────────────────────────────────────────────────┐
│ HOST APP (a normal Android app)                                        │
│                                                                        │
│  UI / launcher        GPU host renderer          container control     │
│  SurfaceView ───────► libOpenglRender.so ─────┐   (start/stop, events) │
│                                               │                        │
│  ┌────────────────────────────────────────────┼─────────────────────┐  │
│  │ GUEST "CONTAINER" (same kernel, own rootfs)│                     │  │
│  │                                             │                     │  │
│  │  init ──► zygote ──► system_server ──► apps │                     │  │
│  │   │         │            │                  │                     │  │
│  │   │         │            └── binder (virtual, userspace)          │  │
│  │   │         └── bionic (guest libc + custom linker/loader)        │  │
│  │   └── virtual root (patched paths)                                │  │
│  │                                                                   │  │
│  │  SurfaceFlinger ──► hwcomposer.<vendor>.so ──► GLES encoder ──────┘  │
│  │  EventHub ◄── virtual input device (UNIX socket / uinput)            │
│  │  HALs ◄── virtual HAL implementations (camera/gps/wifi/sensors/...)  │
│  └──────────────────────────────────────────────────────────────────────┘
└────────────────────────────────────────────────────────────────────────┘
```

The five recurring subsystems:

1. **ROM / rootfs** — a directory tree holding a complete Android system
   (`system/`, `vendor/`, `bin/`, `init`, `init.rc`, …), often shipped
   compressed and sometimes encrypted.
2. **Bootstrap loader / linker** — a custom dynamic linker (or "prelinker")
   that can start a guest `bionic` different from the host's.
3. **Virtual root / container runtime** — how guest absolute paths are made to
   resolve inside the rootfs (patched bionic, or real chroot/namespaces via a
   helper).
4. **Virtualized kernel services** — `binder`, input, graphics, property
   service, `/dev` — all re-implemented in userspace over sockets/pipes.
5. **HAL virtualization** — fake camera, GPS, Wi-Fi, sensors, audio, etc., fed
   from the host or from synthetic data.

---

## 4. Vocabulary used in this research base

| Term | Meaning here |
|------|--------------|
| **Host** | The real phone / the user's Android system. |
| **Guest / container** | The second Android system. |
| **rootfs** | The guest's filesystem tree on the host disk. |
| **goldfish / ranchu** | The official emulator's virtual ARM boards (legacy / current). |
| **AVD** | Android Virtual Device — the emulator's config + system image. |
| **emugen** | AOSP tool that generates GLES/EGL/`renderControl` wire-protocol encoders/decoders. |
| **QEMU pipe / goldfish_pipe** | The emulator's fast host↔guest byte channel (a virtio-like device). |
| **`libOpenglRender`** | The emulator's host-side renderer (decodes GLES, renders on host GPU). |
| **bionic** | Android's libc + dynamic linker (`linker64`). |
| **binder** | Android's IPC; normally a kernel driver (`/dev/binder`). |
| **HWC / gralloc** | Hardware Composer / graphics buffer allocator HALs. |
| **proot** | ptrace-based "fake chroot" used by e.g. Termux/UserLAnd. |
| **prelinker** | A custom first-stage linker that loads the real guest linker. |
