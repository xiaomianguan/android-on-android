# 14 — Activation, Privilege & Security (deep dive)

The single biggest architectural fork between the projects is **how they obtain
privilege**. This doc maps it precisely, because it dictates your feature set,
your install friction, and your attack surface.

---

## 1. The privilege spectrum

```
 pure app (no extra perms)          ADB / Shizuku helper                 root
 ├─ twoyi (rootless container)      ├─ Virtual Master (phantom.killer + shizuku)   ├─ Virtual Master/VMOS Pro on rooted devices
 └─ no activation                   ├─ VMOS Pro (wireless ADB + helper)  └─ twoyi in-guest su only
                                    └─ user enables wireless debugging
```

* **twoyi** proves the **pure-app** endpoint is achievable for a functional
  container.
* **Virtual Master / VMOS Pro** add a privileged helper to do things a plain app cannot
  (and to defeat Android 12+ process limits). The cost is a manual activation
  flow and a security surface.

---

## 2. twoyi — pure rootless app

* No activation, no special permissions beyond storage for ROM import.
* Root is provided **inside the guest**: a static `system/xbin/su`,
  `init.superuser.rc` (`su --daemon`), and `superuser.zip` assets; the app uses
  `libsu`/`Shell` to drive guest-side `su`. So "root" means *root in the
  container*, not on the phone.
* Magisk/Xposed are supported by shipping them into the guest and patching
  `init.rc`.
* Privileged host operations are avoided entirely; the container is built from
  what an app can do (its own data dir + sockets).

---

## 3. Virtual Master — Shizuku/ADB activator (and the real reason it exists)

* `com.phantom.killer.disabler` (`DisablerLauncher.isActivationRequired`) gates
  on SDK ≥ 31. Three methods: `shizuku`, `wireless_adb`, `direct`.
* Bundles a **Shizuku** provider (`rikka.shizuku.ShizukuProvider`, exported).
* Wireless-ADB: mDNS `_adb-tls-pairing._tcp.`, host `127.0.0.1`, connection port
  **5555**, 6-digit code; `AdbConnManager` generates an RSA-2048 key + self-signed
  X.509 (`CN=host`) cached in the app files dir; ADB CNXN payload `host::\0`,
  TLS exporter label `adb-label\0`.
* **Why activation is needed (the key insight):** Android 14+ caps *phantom*
  (child) processes, and a container needs many (`zygote`, `system_server`,
  apps). The unlock pipeline runs over the ADB shell:
  ```
  device_config set_sync_disabled_for_tests persistent
  device_config put activity_manager max_phantom_processes 2147483647
  settings put global settings_enable_monitor_phantom_procs false
  device_config get activity_manager max_phantom_processes
  ```
  i.e. it **disables the phantom-process killer** for the app's uid. On rooted
  devices `su` is used instead.
* On rooted devices it can also use `su`.
* Manifest requests `INSTALL_PACKAGES`, `INTERACT_ACROSS_USERS_FULL`,
  `MANAGE_EXTERNAL_STORAGE`, `DUMP`, `QUERY_ALL_PACKAGES`,
  `SYSTEM_ALERT_WINDOW`, `BIND_VPN_SERVICE`, plus `VMVpnService` exported.

---

## 4. VMOS Pro — wireless ADB + privileged helper (full flow)

This is the most elaborate activation flow and worth documenting precisely
because it shows exactly what a plain app *cannot* do and how it works around it.

1. **Gate checks** (`C9516`): read `Settings.Global`:
   `adb_enabled`, `adb_wifi_enabled`, `development_settings_enabled`. The UI
   walks the user through enabling them; `CoreAccessibilityService`
   ("自动打开无线调试", exported, `BIND_ACCESSIBILITY_SERVICE`) can even toggle
   wireless debugging by driving the Settings UI.
2. **Discovery** (`C9396`/`AdbMdns`): `NsdManager` discovers
   `_adb-tls-pairing._tcp` and `_adb-tls-connect._tcp`.
3. **Pairing**: `WirelessDebugPairService` collects the 6-digit code (via
   notification `RemoteInput`); `moe.shizuku.manager.adb.AdbPairingClient`
   (vendored Shizuku, loads `libadb.so`) performs SPAKE2 pairing
   (`PairingContext.nativeInitCipher/nativeEncrypt`).
4. **Shell**: `AdbShellCmdTool` (`C9510`) opens an ADB shell to
   `127.0.0.1:<port>` using a local ADB key, then (`C9189`):
   ```
   cp <nativeLibDir>/libvmos.so        /data/local/tmp/vmos_starter
   cp <nativeLibDir>/vmos_start_sh.so  /data/local/tmp/vmos_starter_sh.sh
   sh -c /data/local/tmp/vmos_starter_sh.sh
   ```
   `vmos_start_sh.so` (a shell script) re-copies/chmods/chowns (uid 2000) the
   starter and `exec`s it.
5. **Helper**: `libvmos.so` becomes `com.vmos.helper_service` (`setArgV0`) and
   starts a **Netty server on TCP 22681** (`k98`). Its JSON (Gson) protocol:
   `feature=2` start VM, `feature=3` stop VM, `feature=4/5` run an arbitrary
   command via `su`/`sh`/`run-as`, `feature=6` exit, `feature=7` probe.
6. **Root fallback**: `i07` supports `su`; `rs9` runs `su` + `pm install`;
   `tt9` deploys `/data/local/tmp/superuser.apk` and probes root paths.

---

## 5. Why a helper is needed (what an app can't do)

| Need | App | Helper (shell/root) |
|------|-----|---------------------|
| `chroot`/`mount`/namespaces | ✗ | ✓ |
| create device nodes | ✗ | ✓ |
| access `/dev/binder` | ✗ | ✓ |
| spawn many child processes | limited (Android 12+ cap) | ✓ |
| install/run privileged binaries | ✗ | ✓ |
| set SELinux contexts | ✗ | ✓ |

The helper turns "container in an app" into "container with a real runtime".
twoyi avoids the helper by implementing the container purely in userspace
(path translation + socket-backed kernel services).

---

## 6. Security analysis (what to copy and what to avoid)

**Good (reuse):**
* twoyi's **no-privilege** design: nothing to escalate, no attack surface.
* ADB pairing via the **user's own** wireless-debugging credentials (VMOS Pro)
  is legitimate *if* scoped and authenticated.
* VPN via `VpnService` (no root).

**Bad (do not replicate):**
* **Unauthenticated local command server** (VMOS Pro TCP 22681): any local app
  can send `feature=4` and get arbitrary `su`/`sh` execution. Authenticate with
  a per-session token and bind to loopback only.
* **Broadcast/permission abuse**: `INSTALL_PACKAGES`,
  `INTERACT_ACROSS_USERS_FULL`, `DUMP`, exported `VpnService` — minimize.
* **Obfuscated native cores** (`libvm.so`, `libloader.so`): you cannot audit
  them; a reimplementation should be readable.
* **`setArgV0` masquerading** as another process name: used to hide the helper;
  avoid.
* **Encrypting ROMs with a signature-derived key**: security theatre; use a real
  per-install secret or none.

---

## 7. Recommended privilege model for a new tool

1. **Default: rootless** (twoyi-style). Everything works without extra grants.
2. **Optional accelerator: Shizuku** (not raw ADB), so the user grants a
   revocable, auditable capability instead of pasting pairing codes.
3. **Optional: root**, if present, for mount namespaces and device nodes.
4. **Never**: an unauthenticated privileged socket; exported components that
   can be driven by third parties; process-name masquerading.
5. **Always**: bind to loopback, authenticate peers, validate every command,
   and document what the helper can do.
