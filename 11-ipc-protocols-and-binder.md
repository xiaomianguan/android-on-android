# 11 — IPC, Protocols & Binder (deep dive)

Everything that crosses the host/guest boundary is a **protocol**. This doc
enumerates every channel in every project, plus the hardest one — **binder**.

---

## 1. What binder normally is (the thing being virtualized)

Android's IPC is binder: a kernel driver (`/dev/binder`, `/dev/vndbinder`,
`/dev/hwbinder`) implementing an object-reference system. A process:

1. opens `/dev/binder`,
2. `mmap`s a shared region,
3. issues `ioctl(BINDER_WRITE_READ)` carrying commands
   (`BC_TRANSACTION`, `BC_REPLY`, `BC_ACQUIRE`, `BC_INCREFS`, …) and reading
   back `BR_*` events,
4. registers with `servicemanager` via `BINDER_SET_CONTEXT_MGR`,
5. passes **file descriptors** inside transactions (`BC_FREE_BUFFER`,
   `binder_fd` arrays).

The kernel driver is what makes refcounting, death notifications, priority
inheritance and FD passing work. A container with **no guest kernel** must
reproduce all of this in userspace.

---

## 2. Channel inventory

### 2.1 twoyi

| Channel | Type | Purpose |
|---------|------|---------|
| `TWOYI_SOCK` | **abstract** `LocalSocket` (`SOCKET_SEQPACKET`) | guest→host control: `PING`, `BOOT_COMPLETED`, `SWITCH_HOST`, `SETTINGS` |
| `<rootfs>/dev/socket/property_service` | UNIX socket | guest property service (redirected) |
| `<rootfs>/opengles{,2,3}` | UNIX sockets | graphics (see doc 10) |
| `<rootfs>/dev/input/{touch,key0}` | UNIX sockets | input (see doc 13) |
| guest `adbd` on **22122** | TCP | app installs APKs via `libadb.so` (`adb -P 9563 connect localhost:22122`, `install -t -r`) |
| `TwoyiDocumentsProvider` | DocumentsProvider | file exchange between host and guest `sdcard` |

Host-side classes: `TwoyiSocketServer` (binds the abstract socket, reads ASCII
lines) and `TwoyiMessenger` (connects, sends). Boot completion releases a
`CyclicBarrier` in `TwoyiStatusManager`.

### 2.2 Virtual Master

| Channel | Type | Purpose |
|---------|------|---------|
| `<vmDir>/dev/event` | `LocalSocket` (FILESYSTEM) | host↔guest event bus |
| `<vmDir>/dev/input/touch` | UNIX socket | input |
| guest `adbd` via `service.adb.tcp.port` | TCP | guest adb |
| VPN TUN + netlink | `VpnService`/netlink | network + virtual Wi-Fi/cell config |

Event protocol: `DataInputStream.readUTF()` / `DataOutputStream.writeUTF()`
with a **backtick separator** `name`arg`. Actions are
`com.android.vmcore.action.*`:

```
VM_SERVER_READY, ANDROID_OS_READY, BOOT_COMPLETED,
START_INSTALL_APP / INSTALL_APP_RESULT, START_UNINSTALL_APP / UNINSTALL_APP_RESULT,
TASK_STACK_CHANGED, CLIPBOARD_DATA, OPEN_OUTER_PAGE, TOGGLE_RECENTS,
SEND_KEY_EVENT, SET_LOCALE_AND_TIME, SET_GLOBAL_SETTING,
SHOW_NAVIGATION_EVENT, HIDE_NAVIGATION_EVENT, SYNC_PROP_EVENT, KILL_APP,
EXPAND_NOTIFICATIONS_PANEL, COLLAPSE_NOTIFICATIONS_PANEL,
SET_NAVIGATION_BAR_RTL, EXECUTE_COMMAND
```

`BinderService` bridges binder: `setupBinder(vmid, code, 1, 2, name, parcel)`
marshals an `Intent` and calls `bindService` against a component inside the
guest package, after obtaining the `peekService` transaction code by reflection
on the host's `IActivityManager`.

### 2.3 VMOS Pro

| Channel | Type | Purpose |
|---------|------|---------|
| `osimg/socket/ex_engine`, `ex_plugin` | UNIX + abstract + inet | engine/plugin bus (`ExEngineService`) |
| `/dev/socket/socket/ex_engine` | UNIX (guest) | guest side of the bus |
| `rfopengles22501`, `rfserver` | UNIX | graphics |
| `rftouch22801` | UNIX | input |
| `rfaudio` | UNIX | audio |
| `vknel_pipe` | FIFO | kernel/loader pipe |
| `dev/socket/{bcs,bis,bhs}` | UNIX | binder-ish control sockets |
| `start`, `zygote`, `zygote32/64`, `zygote_mgr` | UNIX | process control |
| helper Netty server | **TCP 22681** | privileged command server (JSON/Gson) |
| `_adb-tls-pairing._tcp` / `_adb-tls-connect._tcp` | mDNS | wireless ADB discovery |

`EngineSession` frames messages as **`[4-byte big-endian length][Parcel bytes]`**
(≤ 10 MiB) and **passes file descriptors** via
`setFileDescriptorsForSend(fd, fds, count, out)`. Verbs include install/uninstall
APK, query files, read package path, start intent, volume mute, transfer, clip,
ping.

`libmci.so` carries the media/control protocol ("shouzhiyun" SDK, FlatBuffers +
FFmpeg 3.3.2). Every message is framed by a **6-byte `csproto` header**
`[category:u8][id:u8][len:u32-LE]`; the receiver dispatches on
`type = (category << 8) | id` with the payload at offset 6. Payloads are **raw**
(media/input frames) or **FlatBuffers** (handshake/control/output). Categories:
`0x00` handshake, `0x01` control, `0x02` input/sensors (`InputTouchReq=0x02C8`,
`InputGyro=0x02CC`, …), `0x03` output, `0x04` A/V (`AvAudio=0x0400`,
`AvVideo=0x0401`), `0x05` check-app. Transport is **TCP ± TLS**; the handshake
is an **RSA-2048 OAEP** encryption to a fixed server public key. See doc 05 §7.

The **renderer** channel `%s/rfopengles%d` exchanges **fixed 524-byte (0x20c)
records** `{ int32 cmd; char arg[512]; }` (`rfsendcmd`/`rfrecvcmd`), with bulk
bytes via `rf_send_buffer`, and passes gralloc/EGLImage **file descriptors** over
`%s/NATIVEBUFFERsocket` via `sendmsg`/`recvmsg` (SCM_RIGHTS).

---

## 3. Binder virtualization, three ways

### 3.1 twoyi — a userspace binder *driver* (`libvbinder.so`)

* The guest's `libbinder.so` and `servicemanager` **link `libvbinder.so`**
  (`NEEDED`), so binder calls go to it instead of `/dev/binder`.
* `libvbinder.so` implements the **ioctl surface in userspace** and exports
  exactly `virtual_binder_open`, `virtual_binder_ioctl`, `virtual_binder_mmap`,
  `virtual_binder_driver_main`.
* It implements the full `BINDER_*` ioctl set (`BINDER_WRITE_READ`,
  `BINDER_SET_CONTEXT_MGR(_EXT)`, `BINDER_VERSION`, `BINDER_SET_MAX_THREADS`,
  `BINDER_ENABLE_ONEWAY_SPAM_DETECTION`, `BINDER_FREEZE`,
  `BINDER_GET_NODE_INFO_FOR_REF`, `BINDER_GET_NODE_DEBUG_INFO`), the `BC_*`/`BR_*`
  command set (`BC_TRANSACTION/REPLY/ACQUIRE/RELEASE/INCREFS/DECREFS/
  REQUEST_DEATH_NOTIFICATION/FREE_BUFFER…`, `BR_TRANSACTION/REPLY/DEAD_BINDER/
  SPAWN_LOOPER…`), node/ref bookkeeping, death notifications, transaction
  stacks and fd fixup.
* Transport between processes: **local sockets** —
  `VBINDER_CLIENT_CONNECT`, `VBINDER_CLIENT_READ_WRITE`, `VBINDER_FD_ACCEPT`,
  `VBINDER_FD_CONNECT`, `VBINDER_IOCTL_CONNECT`, `VHWBINDER`
  (`recvSeqPacket`/`sendSeqPacket`), with SCM_RIGHTS fd passing.
* Shared memory: `/dev/ashmem` **and** `memfd_create` (+ `F_ADD_SEALS`).
* Virtual device nodes `/dev/vbinder`, `/dev/vndbinder`, `/dev/hwbinder` are
  translated to `<rootfs>/dev/…`; plain `/dev/binder` is passed through to the
  host kernel.
* This is the most "complete" binder emulation: it is a drop-in driver.

### 3.2 Virtual Master — binder **injection** into the guest

* `BinderService.setupBinder(...)` (native) marshals an `Intent` and injects a
  binder into the guest's `ActivityManager` via `bindService`; the
  `peekService` transaction code is read reflectively from the host's
  `IActivityManager`.
* `getBinderVersion()` is a health check.
* The guest then runs its own `servicemanager` for its own binder domain; the
  host/guest boundary is crossed through the event bus + injected binder rather
  than a shared driver.

### 3.3 VMOS Pro — control sockets + delegated processes

* `dev/socket/{bcs,bis,bhs}` (binder/interface/system sockets) plus the
  `ExEngineService` bus for host↔guest operations.
* `NativeUtil.startzygote_d*` spawns delegated zygotes in up to 100 `:mN`
  service processes (a workaround for the Android 12+ child-process limit).

### 3.4 Comparison

| | twoyi | Virtual Master | VMOS Pro |
|---|---|---|---|
| binder model | userspace driver (`libvbinder`) | injection + guest servicemanager | control sockets + delegated zygotes |
| FD passing | via libvbinder sockets | via event bus/parcel | via `EngineSession` (SCM_RIGHTS) |
| shared memory | `/dev/ashmem` | ashmem | ashmem / memfd |
| notable | fully self-contained | reflection-based transaction code | 100-service process model |

---

## 4. Designing your own binder bridge (clean-room)

Options, in increasing order of fidelity:

1. **Control-socket bus only** (simplest): keep host and guest binder domains
   separate; cross the boundary with an explicit, typed socket protocol
   (install app, launch, clipboard, keys). This is what most features need.
2. **Injected binder** (Virtual Master): reuse the guest's own `servicemanager` and inject
   a host binder object into the guest framework.
3. **Userspace binder driver** (twoyi): implement the `/dev/binder` ioctl
   surface over a socket, with ashmem and FD passing. Highest fidelity, most
   work; base it on the public binder UAPI (`linux/android/binder.h`) and
   `frameworks/native/libs/binder`.

For a first version, (1) plus a minimal `servicemanager` shim is enough to boot
and install apps; add (3) later if you need full framework integration.

---

## 5. Security notes (from the field)

* VMOS Pro's helper server on **TCP 22681** accepts JSON commands
  (`feature=4` = run arbitrary command via `su`/`sh`/`run-as`) with **no
  authentication** — any local app could talk to it. Do not replicate.
* ADB-based activation means the guest can be reached over loopback adb; bind
  to loopback and authenticate.
* The event buses carry powerful actions (`EXECUTE_COMMAND`, `KILL_APP`,
  `SET_GLOBAL_SETTING`); validate the peer and arguments.
