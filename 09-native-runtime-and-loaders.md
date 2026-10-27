# 09 — Native Runtime & Loader Layer (deep dive)

This is the layer that makes "a second Android userspace on the host kernel"
possible: the **bootstrap loaders**, the **guest linkers**, the **hooking
engine**, and the **obfuscation** each vendor applies. This is the most
important subsystem to get right in a reimplementation.

---

## 1. Why a custom loader is unavoidable

On a phone there is exactly one dynamic linker the kernel knows: the host's
`/system/bin/linker64`. The guest's `bionic` (libc + linker) is a *different
build*, often a *different Android version*, and the guest's `init` assumes a
root filesystem. So the bootstrap must answer three questions:

1. **How does the first guest process run?** (statically-linked `init`, or a
   shipped interpreter)
2. **How does the guest's own linker get loaded?** (a "prelinker" / loader that
   maps it and transfers control)
3. **How do the guest's absolute paths and library searches resolve inside the
   rootfs?** (patched bionic, or a runtime path translator)

Each project answers differently. Below is the full comparison.

---

## 2. twoyi — static `init` + `libloader.so` + patched bionic

```
app (uid u0_aN)
  └─ exec "./init" (STATIC, ET_EXEC, no PT_INTERP)  cwd=/data/data/io.twoyi/rootfs
        env TYLOADER=/data/data/io.twoyi/loader64   (symlink → libloader.so)
     │
     ├─ init bootstraps the guest (ueventd, servicemanager, zygote, …)
     └─ guest dynamic binaries use the patched linker64/libc.so
```

* **`init` is statically linked** — `readelf` shows `Type: EXEC`, no `PT_INTERP`
  and no `NEEDED`. It is the only guest program the host kernel can start.
  It contains the capability-name table (`CAP_SYS_*`), the `TYLOADER` and
  `INIT_SECOND_STAGE` env handling, and the hard-coded rootfs/loader paths.
* **`libloader.so`** (51 KB) is the "loader" referenced by `TYLOADER`:
  * `INTERP /system/bin/linker64`; `NEEDED libc.so, libdl.so`.
  * Imports: `dlopen, dlsym, dlclose, dlerror, mmap, mprotect, munmap, open,
    read, lseek, sprintf, __libc_init, _exit, __cxa_atexit, memcpy, memset`.
  * Its `_start` calls `__libc_init` normally, but the body is **obfuscated**
    with long mixed-boolean-arithmetic (MBA) sequences — deliberately closed.
  * **Behaviour recovered by tracing** (Unicorn; `trace_libloader2.py`): its
    `main` is a **minimal ELF loader** — it takes the program path, prepends the
    rootfs (`open("/data/data/io.twoyi/rootfs" + argv0)`), `mmap`s each `PT_LOAD`
    segment at its vaddr, `mprotect`s it, and jumps into the entry point. It does
    **not** do path translation (that lives in the guest `libc`/`linker64`); it
    only loads and starts the guest ELF. Traced sequence:
    `open(/data/data/io.twoyi/rootfs/init)` → `mmap(0x400000, 0x187000)` →
    `mprotect` → `mmap(0x537000, 0x50000)` → jump.
  * The app creates `<dataDir>/loader64` (and a `loader32` counterpart for
    32-bit guests) as symlinks to it.
* **The guest's `libc.so` and `linker64` are patched with a path-translation
  layer** (see `03-twoyi.md` §4): `translate_path` prefixes absolute paths
  with `/data/data/io.twoyi/rootfs`; `is_need_translate` passes through `/proc`,
  most of `/dev` and `/sys`; `translate_socket` rewrites UNIX socket addresses.
  Both binaries are **not stripped**, so this is fully readable — the cleanest
  reference for a rootless virtual root.
* **`execve` is patched to re-exec 64-bit ELFs through the loader**: it reads
  `TYLOADER=` (default `/data/data/io.twoyi/loader64`) and calls
  `real_execve(loader, argv, envp)`. So the loader is invoked not only for the
  first `init` but for every 64-bit ELF the guest execs.
* **`libc.so` also carries an inline-hook framework** (`And64InlineHook.cpp`) —
  twoyi's native-hook capability.
* **UID/GID are virtualized** via a `/dev/vid/<pid>` pseudo-file (28-byte
  struct), so guest processes appear to have their own uids while really running
  as the host app uid.

**Deobfuscation status:** `libloader.so` internals unresolved (MBA, no external
call graph to trace statically). Its role and interface are unambiguous.

---

## 3. Virtual Master — `libkrloader64.so` → `libkr64.so` (linker + hook engine)

```
app (uid u0_aN)
  └─ System.loadLibrary("vm") → libvm.so (OLLVM-obfuscated)
       └─ NativeHelper.startOS(vmid, osVersion, romVersion, <dataDir>/lib64)
            ├─ creates <dataDir>/lib64 → nativeLibraryDir
            └─ libkrloader64.so            INTERP /system/bin/linker64
                 │  (host loader; _start → internal init at 0x4024; obfuscated)
                 ▼
               libkr64.so                  INTERP <dataDir>/lib64/libkrloader64.so
                 │  (PATCHED BIONIC LINKER + ShadowHook)
                 ▼
               guest init → zygote64 → system_server → apps
```

* **`libkrloader64.so`** (234 KB, 32-bit sibling `libkrloader32.so`): the host
  loader. `_start` at `0x2cd0` does `mov x0, sp; bl <init>; br x0` and the rest
  is an OLLVM state machine; strings are encrypted. Built for `marlin`
  (`.../krloader_intermediates/LINKED/libkrloader64.so`). Uses
  `/dev/__properties__`, `/dev/socket/{logdw,property_service}`, `mount`.
* **`libkr64.so`** (1.6 MB) is the **guest linker** (a modified `bionic`
  `linker64`), and it is *not* just a linker — it bundles **ShadowHook 2.0.1**,
  an inline-hook framework:
  * Exports `__dl___linker_init`, `__dl__Z8__dlopenPKciPKv`,
    `__dl__Z9do_dlopen…`, `dlopen_ext`, `__loader_dlopen`,
    `android_dlopen_ext`, `soinfo::call_constructors`,
    `soinfo::call_destructors`.
  * **Hooks `soinfo::call_constructors` / `call_destructors`** ("hook(invisible)
    soinfo::call_constructors") — i.e. it intercepts every library load to run
    Virtual Master code (for its virtual-location / app-monitoring / multi-account
    features).
  * To find the right offsets it does a **`soinfo` memory scan** at runtime
    ("linker: soinfo memory scan pre/post OK … load_bias … name … phdr …") —
    because the guest linker version differs from the build target.
  * Uses `libshadowhook.so` (`shadowhook-enter`, `shadowhook-hub-trampo`,
    `shadowhook-island`, "hook (with island) OK") for inline hooks, and
    references `/system/bin/app_process64`, `/system/bin/linker64`,
    `/system/build.prop`, `/system/lib64/liblzma.so`.
* **Obfuscation:** `libvm.so` (9.7 MB) and `libkrloader64.so` are OLLVM-protected:
  ~110 `.datadiv_decode*` functions (global-data obfuscation) plus **runtime
  string encryption** (most `/…` and property strings are ciphertext on disk).
  `libkr64.so` is much less obfuscated (readable linker/shadowhook strings).

> **Takeaway for a reimplementation:** Virtual Master's `libkr64.so` shows the "right"
> shape for a guest linker — a bionic `linker64` build plus a well-defined
> hook/interposition point at `soinfo::call_constructors`. Use AOSP `bionic`,
> and if you need hooks, use a maintained library (ShadowHook/ByteHook) rather
> than writing inline patching yourself.

---

## 4. VMOS Pro — prelinker → linkers → kernel loader

```
app (com.vmos.pro)
  └─ libkrnloader.so --file <vmDir>/libvmoskernel.so
        │  (32-bit ARM; strings: "libos_loader",
        │   "loader : can't mmap addr[0xB0000000], tray again!",
        │   "loader : dlsym = %s res = %p", "loader : main in...")
        │  maps the guest image at a fixed base (0xB0000000) and execve()s it
        ▼
     libvmoskernel.so  ("guest kernel")
        └─ guest init  → its interpreter is a shipped linker:
             system/bin/initlinker64  →  libinitlinker64.so
        │
   libinitlinker64.so / libylinker64.so   INTERP = /data/data/com.vmos.pro/vmb/libprelinker64.so
        │  (guest linkers; libylinker exports call_func, jmp_syscall_handler)
        ▼
   libprelinker64.so   INTERP = /system/bin/linker64
        │  (a full bionic linker built as target "prelinker64", product marlin)
        │  exports: __linker_init, _start, _start_thread, _syscall,
        │           linkerload_elf_binary_execv, copy_image_memory,
        │           phdr_table_get_load_size, init_seccomp,
        │           install_sigsys_handler, install_sigsegv_handler
        ▼
   guest bionic userspace
```

* The **prelinker** is the key: it is a bionic linker repackaged as the
  interpreter of the guest linkers. `__linker_init` parses the auxv for
  `AT_BASE`, maps the linker at fixed addresses, and returns the guest entry.
  It installs a **seccomp-BPF filter** (`init_seccomp`: `PR_SET_NO_NEW_PRIVS` +
  an arch check + a syscall whitelist) and a **SIGSYS handler that re-issues
  blocked syscalls in userspace** (notably `mmap=222`, `rt_sigaction=134`) —
  which is how the guest is allowed to `mmap` at the fixed base `0xB0000000`.
  `linkerload_elf_binary_execv` is a custom ELF loader.
* `libkrnloader.so` is only 5.6 KB (32-bit **Thumb-2** `main`): it parses
  `--file <rom>/libvmoskernel.so`, `dlopen`s it, `dlsym("main")`, calls it, and
  if the guest kernel's `main` returns 37/65 (fixed-`mmap` failure) it
  **`execve`s itself to retry**.
* `libylinker64.so` exports `call_func` (a **magic trampoline** to
  `0x1234567890ABCDEF`, patched at load to the real function) and
  `jmp_syscall_handler` (a **user-space `svc` replacement** that packs the
  syscall registers and forwards to a dispatcher with tag `999`).
* `libnative-lib.so` confirms the layout in strings:
  `%s/osimg/r/ot%s/libvmoskernel.so`,
  `%s/osimg/r/ot%s/system/bin/initlinker32|64`,
  `--socket-name=zygote`, `ANDROID_SOCKET_zygote=%d`, `/dev/binder`,
  `LOOP_MOUNTPOINT`.

**Deobfuscation status:** the linker chain is largely readable (symbols +
strings); the exact seccomp policy and the fixed mapping need a runtime trace
to fully enumerate.

---

## 5. Loader-chain comparison

| | twoyi | Virtual Master | VMOS Pro |
|---|---|---|---|
| First guest process | static `init` | loader → guest linker | kernel loader → guest "kernel" |
| Loader | `libloader.so` (MBA-obfuscated) | `libkrloader64.so` (OLLVM) | `libkrnloader.so` (5.6 KB) |
| Guest linker | patched `linker64` | patched bionic `linker64` (`libkr64.so`) + ShadowHook | `libylinker64`/`libinitlinker64` |
| Prelinker | none | none | `libprelinker64.so` (bionic linker) |
| Path virtualization | patched `libc`/`linker64` (`translate_path`) | via loader/guest linker + `/lib64` symlink | via prelinker + `LOOP_MOUNTPOINT` |
| Hooking | none built in | ShadowHook (ctors/dtors) | not observed in linkers |
| Obfuscation | MBA on loader; rest readable | OLLVM on `libvm`/loader; string encryption | mostly readable |

---

## 6. `bionic` changes you will need (clean-room)

If you build your own guest linker from AOSP `bionic`:

1. **Static `init`** — compile the guest `init` statically (or ship a tiny
   static bootstrap) so it can be the first process.
2. **Path translation** — add a small, explicit translation layer to `libc`
   (`openat`-family) and the linker (`open_library_on_paths`), with a
   passthrough list for `/proc`, `/sys`, `/dev`, and SELinux paths. This is the
   twoyi approach and is fully readable.
3. **Library search paths** — point the linker's default search paths at the
   rootfs (`/…/system/lib64:/…/vendor/lib64`).
4. **Namespace isolation** — optionally use the linker namespace API to keep
   guest libs separate from host libs.
5. **Hooks (optional)** — integrate a maintained inline-hook library rather
   than patching `soinfo` by hand.

Avoid twoyi's hard-coded absolute paths and Virtual Master's heavy obfuscation: make the
loader **configurable** (read the rootfs/loader paths from an env var or a
config file) so ROMs stay relocatable.
