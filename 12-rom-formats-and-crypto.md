# 12 — ROM Formats, Packaging & Crypto (deep dive)

How each project stores, verifies, extracts and (sometimes) encrypts the guest
system. This determines your build/packaging pipeline.

---

## 1. The shape of a "ROM"

A ROM is not a disk image in any of these projects — it is a **directory tree**
of a complete Android system, packaged for transport. On disk after install it
is always an unpacked tree owned by the app.

| | twoyi | Virtual Master / VMOS Pro |
|---|---|---|
| Packaged as | `assets/rootfs.7z` | a ROM package (zip) with `manifest.json` |
| Members | the whole tree under `rootfs/` | `vm_config.xml`, `rom.zip`/`rom.img`, `superuser.zip`, `xposed.zip`, `magisk.zip`, `play.zip`, `data.zip` |
| Metadata | `rootfs/rom.ini` (`author`, `code`, `version`, `md5`) | `manifest.json` (`id`, `display_name`, `os_version`, `rom_version`, `rom_uri[]`, `su_uri`, `magisk_uri`, `xposed_uri`, `play_uri`) |
| Integrity | `md5` in `rom.ini` | per-`Asset` md5 |
| Encryption | none | **AES** on `rom.img`/`rom1.img` (`?e=x&c=v2`) |
| Extra format | — | VMOS Pro **packed inode filesystem** (`libvmkrnfs.so`) |

### 1.1 The supplied sample (Virtual Master / Virtual Master)

`subjects/rom_11_0_0_1036_3280.zip` is a **Virtual Master / Virtual Master Android
11 ROM**:

```json
{ "id":"rom_default_11.0.0", "display_name":"Android 11.0.0",
  "rom_version":1036, "os_version":11, "support_a32":false, "support_a64":true,
  "minimum_sdk_int":30, "minimum_app_ver":"3.2.19",
  "rom_uri":["rom.img?e=x&c=v2", "rom1.img?e=x&c=v2"],
  "magisk_uri":"asset:///plugins/magisk.zip?h=true",
  "su_uri":"superuser.zip?h=true", "xposed_uri":"",
  "play_uri":"asset:///plugins/play.zip" }
```

**RESOLVED — the images are XOR-encrypted 7z archives.** `rom.img`/`rom1.img`
are **XOR with the 16-byte repeating key `%z89aviCM0KkbEs9`**, yielding a **7-Zip
archive** (LZMA + BCJ) of a full filesystem tree. Proof: the XOR output starts
with `37 7A BC AF 27 1C 00 03` and the 7z **StartHeaderCRC validates**. `7z l`
then lists 2479 files (`fs/system/…`) + 1440 files (`fs/vendor/…`).

* `e` = encryption (`x` = XOR), `c` = container (`v2` = 7z/LZMA).
* The header leaks the key where the plaintext is zero (`%z89FviCM0Kk` at 0x10).
* The app-side transform is `m5/WWWWoWWWWo.java` (a `w4` XOR stream when
  `param("e")=="x"`, else AES with the same key); 7z extraction uses `libun7z`.
* `superuser.zip` (`?h=true`) is **AES-ECB** with the same key → a normal zip
  containing the **ChainsDD Superuser overlay** (`system/app/Superuser/
  Superuser.apk`, `system/xbin/daemonsu`).
* The decrypted guest is a **hybrid Android 11 image** — an AOSP **GSI** system
  side plus a **goldfish/emulator** vendor side, dressed up as a Pixel XL
  (`marlin`) with Pixel-4 (`flame`) board props. See
  `19-virtual-master-and-rom.md`.

---

## 2. twoyi packaging

* ROM shipped as `assets/rootfs.7z` (~200 MB compressed → ~612 MB extracted).
* `RomManager`:
  * copies the asset to `<files>/rootfs.7z`,
  * extracts with `P7ZipApi.executeCommand("7z x -mmt=<n> -aoa … -o<dataDir>")`,
  * reads `rom.ini` for version/code to decide upgrades,
  * supports a 3rd-party ROM (`rootfs_3rd.7z`) and factory reset,
  * writes `vendor/default.prop` (locale, timezone, `ro.sf.lcd_density`).
* The extracted tree is `/data/data/io.twoyi/rootfs` — and because the ROM has
  that path baked into ~11 binaries, **relocating requires in-place patching**
  (the threetwi `rom_patcher.rs` automates it; rootfs ≤ 26 chars, loader ≤ 28).
* `rootfs/rom.ini` example (twoyi 0.5.4): `author=weishu`,
  `code=20220321182552`, `version=1.0.0-…`.

---

## 3. Virtual Master / VMOS Pro pipeline

1. **Fetch**: download the package (or use `asset:///plugins/…` for the
   bundled magisk/play/superuser/xposed).
2. **Verify**: md5 each `Asset`.
3. **Unpack**: zip via blankj `ZipUtils`; 7z via `libun7z`; images decrypted by
   the native layer.
4. **Install**:
   * Virtual Master: `<externalFilesDir>/rom/<rom_id>_<rom_version>/`, then staged under
     `<dataDir>/vm/vm<N>/fs`.
   * VMOS Pro: `<dataDir>/osimg/r/ot%02x/` (per VM), with `guestOSInfo` (JSON
     `RomInfo`: `androidVersion`, `supportAbis`, sensor flags, …).
5. **Overlays**: `SuperuserTask`, `MagiskTask`, `XposedTask`, `GooglePlayTask`
   copy the plugin zips into the guest's `system/` and patch `init.rc`.

`RomInfo` drives compatibility: `support_a32`/`support_a64`,
`minimum_sdk_int`, `minimum_app_ver`.

---

## 4. Crypto: `libcore.so` (VMOS Pro)

`libcore.so` (224 KB) is the crypto/integrity core:

* **AES** (`AES_init_ctx`, `AES_CBC_encrypt_buffer`, `AES_CBC_decrypt_buffer`,
  `AES_CTR_xcrypt_buffer`, `AES_ECB_*`), **MD5**, **PKCS7**
  (`pkcs7_parse_pkcs7`, `pkcs7_get_from_apk`), and a zip reader/writer.
* JNI (`com.vpi.core.utils.VpiNativeUtils`):
  `initData(sourceDir)`, `getFileMD5(path)`, `encryptData(ctx,path)`,
  `decryptData(ctx,path)`, plus `unseal`, `xor_data`, `getAppPath`,
  `getProcessName`, `getSignatureMD5`.
* `initData(applicationInfo.sourceDir)` initializes the crypto with the **APK
  path**; `getSignatureMD5` / `getFileMD5` gate integrity.

**What `libcore` actually does (resolved):**

* `initData(apkPath)` parses the APK's **PKCS#7 signing block** and returns the
  **hex MD5 of the signing certificate** (16 bytes). `getSignatureMD5` yields the
  same 16 bytes.
* `encryptData`/`decryptData` are **XOR with that 16-byte signature MD5 +
  Base64** — used for **API-response/string obfuscation**, *not* ROM encryption
  (confirmed in Java: `new String(VpiNativeUtils.decryptData(ctx, msg))`).
* The exported **tiny-AES** (`AES_CBC/CTR/ECB`) has **no in-binary caller and no
  importing library**, so it is not on the ROM path either.
* `unseal` is a hidden-API/non-SDK **restriction bypass** (matches the bundled
  `org.chickenhook.restrictionbypass`).
* `getFileMD5`/`getSignatureMD5` are integrity tokens.

**ROM images (honest status):** `rom.img`/`rom1.img` have entropy 8.0 and a
**structured common prefix** (first 8 bytes identical; `%z89FviCM0Kk` at 0x10;
long identical runs), characteristic of a **stream cipher (e.g. AES-CTR) over a
compressed image**. The key is **not in the host APK's native libraries** —
likely in the guest `libvmoskernel.so` or server-side. `?e=x&c=v2` tags the
scheme. (The `%z89FviCM0Kk` prefix resembles Virtual Master's hard-coded report key
`%z89aviCM0KkbEs9`.)

> For a reimplementation: use a **per-install random key stored in the
> Keystore**, or don't encrypt at all (an unpacked rootfs is already readable by
> the app). Encryption mainly complicates your own tooling.

---

## 5. The packed inode filesystem (`libvmkrnfs.so`)

VMOS Pro adds a compact container format for shipping/overlaying a rootfs:

* JNI/API: `init(path, path2, mode)`, `pack()`, `unpack()`,
  `addFile/addDir/modFile/modDir/rmFile/rmDir`, `getInodeByPath`.
* `Inode` = 11 little-endian ints: `mode, owner, group, linkCount, createTime,
  modifyTime, accessTime, dataSize, dataOffset, crc32Hash, xattrSize`.
* So the tree is stored as a flat table of inodes + a data blob, with CRC32 per
  file and xattr support — a tar/erofs-like format tuned for overlays
  (`diff.conf`, per-file deltas).

> For a reimplementation, a plain `tar`/`zstd` (or `erofs`) is simpler and
> tool-friendly; you do not need a custom inode format unless you want
> incremental overlays.

---

## 6. ROM relocation & patching

Because twoyi bakes absolute paths into the guest, ROMs are **path-fragile**.
The community patcher enumerates the exact strings to rewrite:

```
/data/data/io.twoyi/rootfs                                  (26)  base
/data/data/io.twoyi/loader64                                (28)  loader
/data/data/io.twoyi/rootfs/opengles{,2,3}                         graphics sockets
/data/data/io.twoyi/rootfs/dev/socket/property_service            property
/data/data/io.twoyi/rootfs/vendor/lib64/egl/libGLESv{1,2}_CM_emulation.so
```

A clean implementation avoids this entirely by reading the rootfs path from
configuration at runtime (env var / config file) and translating paths in the
container runtime rather than in the binaries.

---

## 7. Repackaging a ROM (practical checklist)

1. Start from a **goldfish/ranchu system image** matching the guest version.
2. Add the container pieces: patched bionic (path translation), a custom HWC
   (`hwcomposer.<vendor>.so`), a userspace binder, virtual input, and a
   materialized `/dev`.
3. Strip or neutralize kernel-dependent services (mounts, cgroups, device
   nodes) in `init.rc`/`init.<board>.rc`; keep the framework services.
4. Ship as a **plain, verifiable archive** with a manifest (id, version, abi,
   min-sdk) and a checksum; make the rootfs path a runtime parameter.
5. Provide a `rom.ini`-style metadata file and support 3rd-party ROMs.
