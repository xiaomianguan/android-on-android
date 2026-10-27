# 16 — Obfuscation, Anti-Analysis & Deobfuscation Notes

Every closed project here protects its native core. This doc records exactly
what protection is used, what it hides, and what was successfully recovered —
so a reimplementation knows what to expect and (importantly) what *not* to do.

---

## 1. Protection map

| Binary | Protection | Readable? | Notes |
|--------|-----------|-----------|-------|
| twoyi `libloader.so` | Inline **MBA** (mixed boolean arithmetic) control flow; stripped | No (logic) | Interface/imports readable; body obfuscated |
| twoyi guest `libc.so`, `linker64` | **none** (only patched) | **Yes** | Fully symbolised — best reference for path translation |
| twoyi `libOpenglRender.so` | none | Yes | AOSP code, patched transport |
| Virtual Master `libvm.so` (9.7 MB) | **OLLVM**: ~110 `.datadiv_decode*` + **runtime string encryption** | Partly | Paths/HAL names readable; most strings ciphertext |
| Virtual Master `libkrloader64.so` | OLLVM (state-machine + string encryption) | No | `_start` → obfuscated init |
| Virtual Master `libkr64.so` | **none** | **Yes** | Patched bionic linker + ShadowHook, fully symbolised |
| VMOS Pro `libprelinker64.so` | none | Yes | bionic linker, symbols present |
| VMOS Pro `libylinker64.so`/`libinitlinker64.so` | none | Yes | `call_func`, `jmp_syscall_handler` |
| VMOS Pro `libvmtools.so`/`libvmkrntools.so` | none | Yes | Emulator renderer symbols present |
| VMOS Pro `libmci.so` | none | Yes | Protocol function names present |
| VMOS Pro `libcore.so` | none | Yes | Crypto symbols present |
| Virtual Master `Java` (dex) | **XOR string obfuscation** (repeating key) + name mangling | Recovered | See §3 |
| VMOS Pro `Java` (dex) | name mangling / `defpackage` | Mostly readable | Strings not XORed |

**Takeaway:** the *most* obfuscated component in each project is the **loader**;
the *least* obfuscated is the **graphics stack** (it is AOSP code). The
mechanisms you most need (graphics, binder surface, paths) are the easiest to
read.

---

## 2. Virtual Master native deobfuscation (`.datadiv_decode`)

Virtual Master's `libvm.so` and `libkrloader64.so` are built with **Obfuscator-LLVM**:

* **Data obfuscation** — ~110 functions named `.datadiv_decode<random>` run from
  `.init_array` and XOR global data with per-region constants.
* **String encryption** — most string literals are stored as ciphertext and
  decoded at use-time (so `strings` shows garbage for them).

I wrote an emulator for the data-div functions
(`workspace/notes/deobf_datadiv.py`) using **Unicorn**:

```
map all PT_LOAD segments at their vaddrs
for each .datadiv_decode* symbol: run it with a stack and a sentinel LR
dump the segments back out
```

Result: **110/110 functions ran with 0 errors**, but the recovered data was
mostly **integer constants, not a string table** (+~600 mostly-binary
"strings"). This confirms the data-div layer obfuscates *constants*; the
*strings* are protected by a separate use-time decryptor.

A scan of `libvm.so`'s `.text` shows the decryptor uses **NEON**: thousands of
`movi v0.16b, #0xNN` + `eor vX.16b, vX.16b, v0.16b` (16-byte XOR with a
per-block constant byte), writing back to `.data` (entropy ≈ 7.0; the encrypted
strings live there). There is **no scalar `ldrb`/`eor`/`strb` loop**, so a
simple XOR-loop scan finds nothing.

**But it was broken anyway.** The strings are XOR-encrypted with a **per-string
single-byte constant**, so a brute-force over all 256 constants per string
(keeping the all-printable, best-scoring result) recovers them. See
`workspace/notes/libvm_strings.py` → **540 decoded strings**
(`workspace/notes/virtualmaster_libvm_strings.txt`). The engine is now readable
(VPN/TUN + lwIP, netlink, Wi-Fi/HAL forwarders, input, audio, display, binder) —
see `21-libvm-engine.md`.

**Still opaque:** the JNI registration tables and class-name strings are built
at runtime (BSS + a separate decode), so the Java↔native method *table* is not
statically recoverable.

---

## 3. Virtual Master Java deobfuscation (XOR)

The Virtual Master dex XORs strings with a repeating key:

```java
public static String m14506WWWWWWWW(byte[] data, byte[] key) {
    for (i…) data[i] ^= key[i % key.length];
    return new String(data, UTF_8);
}
```

A deobfuscator was built (`workspace/notes/virtualmaster_deobf.py`) that scans the jadx
sources for decoder call sites, extracts `(data, key)` pairs and decodes them,
producing **11,854 decoded strings**
(`workspace/notes/virtualmaster_strings_decoded.txt`). Highlights recovered:

* **Event protocol** (full list): `com.android.vmcore.action.{VM_SERVER_READY,
  ANDROID_OS_READY, BOOT_COMPLETED, START_INSTALL_APP, INSTALL_APP_RESULT,
  START_UNINSTALL_APP, UNINSTALL_APP_RESULT, TASK_STACK_CHANGED, CLIPBOARD_DATA,
  OPEN_OUTER_PAGE, TOGGLE_RECENTS, SEND_KEY_EVENT, SET_LOCALE_AND_TIME,
  SET_GLOBAL_SETTING, SHOW/ HIDE_NAVIGATION_EVENT, SYNC_PROP_EVENT, KILL_APP,
  EXPAND/COLLAPSE_NOTIFICATIONS_PANEL, SET_NAVIGATION_BAR_RTL, EXECUTE_COMMAND}`.
* **Service bring-up log tags**: `[doStartOS] os starting`,
  `[setupBinder]`, `[setupInput]`, `[setupAudio]`, `[setupHAL]`,
  `[setupRender]`, `[setupNetlink]`, `[setupVpn]`, `[setupBridge]`.
* **Paths**: `/data/data/…/vm/vm<N>/fs`, `/dev/event`,
  `/dev/{__properties__,__kmsg__,kmsg,dm-user,vmproc,netlink_client,tun.ctrl_client}`,
  `/dev/socket/com.koushikdutta.superuser.daemon`.
* **Guest HALs**: `camera.ranchu`, `gralloc.ranchu`, `vulkan.ranchu`,
  `audio.primary.ranchu`, `fingerprint.ranchu`, `vibrator.ranchu`,
  `sensors@2.1-impl.ranchu`, and notably **`libqemupipe.ranchu.so`**.
* **UI shims**: `libui10.so`, `libui51.so`, `libhostlibui.so`.
* **Magisk/Xposed**: `/data/adb/magisk/*`, `libxposed_art.so`.
* **Network**: `VpnTunThread`, `init tun loop`, `ifname=`, `mac=`,
  `/dev/tun.ctrl_client`, `/dev/netlink_client`.
* **Endpoints**: `api.virtualmaster.app` (the vendor's backend),
  `https://virtualmaster.app`.
* **Vendor**: Virtual Master is operated by "VirtualMaster" (`virtualmaster.app`).

---

## 4. twoyi `libloader.so` (MBA)

* 51 KB, stripped, `INTERP /system/bin/linker64`.
* `_start` calls `__libc_init` normally; the body is a long sequence of
  `mov/movk/bic/eor/orn` operations computing values (MBA).
* The `.rodata` contains only `/system/bin/linker64`, so there is no string
  table to decrypt — the *logic itself* is obfuscated.
* **Not deobfuscated, but its behaviour was recovered by tracing.** Running its
  `main` in Unicorn with stubbed imports (`trace_libloader2.py`) shows it is a
  minimal **ELF loader**: `open(rootfs + program)` → `mmap` the `PT_LOAD`
  segments → `mprotect` → jump. Its role (first-stage loader), interface
  (`INTERP`, imports `dlopen/dlsym/mmap/open/read/sprintf/__libc_init`) and the
  `TYLOADER` reference are fully established.
* For a reimplementation, write your own loader from AOSP `bionic`; you do not
  need this binary.

---

## 5. What obfuscation tells you (design guidance)

1. **Obfuscating the loader is a maintenance dead-end.** twoyi's loader is
   closed and unusable for forking; the project was eventually discontinued
   partly for such reasons. Keep loaders readable.
2. **The valuable IP is the integration, not the loader.** The graphics stack,
   binder surface and HAL interfaces are all public; the loader is a small,
   well-understood piece (map an ELF, set a namespace, jump).
3. **String encryption buys little.** Virtual Master's Java XOR and native use-time
   decryption were both recoverable, and they made debugging harder for the
   vendor too.
4. **Do not ship anti-analysis in a tool users must trust.** A VM tool runs
   untrusted guest code; your own components should be auditable.

---

## 6. Reproducing the deobfuscation

```bash
# Virtual Master native data-div (Unicorn)
python3 workspace/notes/deobf_datadiv.py workspace/apk/virtualmaster/lib/arm64-v8a/libvm.so libvm.so.dec

# Virtual Master Java XOR strings
python3 workspace/notes/virtualmaster_deobf.py        # writes virtualmaster_strings_decoded.txt

# Symbol-level analysis (readable components)
llvm-nm  -D --defined-only <lib.so>
llvm-objdump -d --no-show-raw-insn <lib.so>
strings -t x <lib.so> | grep <needle>
```
