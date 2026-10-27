# 21 — Virtual Master's `libvm.so` Engine (deobfuscated)

`libvm.so` (9.7 MB) is Virtual Master's whole host-side VM engine — the Java
layer is orchestration only. It was previously opaque (OLLVM + encrypted
strings). This doc records the **successful string deobfuscation** and the
engine's internals.

Sources: `workspace/apk/virtualmaster/lib/arm64-v8a/libvm.so`,
`workspace/notes/virtualmaster_libvm_strings.txt` (540 decoded strings),
`workspace/notes/libvm_strings.py` (the decoder),
`workspace/jadx/virtualmaster/` (the Java layer).

---

## 1. The string obfuscation and how it was broken

`libvm.so` stores most of its strings **XOR-encrypted with a per-string
single-byte constant**, decoded **inline at use time with NEON**:

```
ldr   d0, [x14]              ; load encrypted bytes from .data
movi  v2.8b, #0xNN           ; the per-string constant
eor   v12.8b, v0.8b, v2.8b   ; decrypt
```

The encrypted strings live in the `.data` segment (`0x931240`–`0x941b68`); some
strings (paths, `VMCORE`, `vmcore_process`, `ro.build.version.sdk`,
`initOpenGLRenderer`) are stored **plaintext**. Because each string has its own
constant, a naïve `strings` dump shows ciphertext.

**Recovery method** (`libvm_strings.py`): for every maximal printable run in the
`.data` segment, try all 256 single-byte constants, keep the result that is
all-printable and scores best (letters, spaces, structural chars `/._%:()`,
known substrings, minus 3-char repeats), preferring `k=0` (plaintext). This
recovered **540 strings** — the engine is now readable.

> The JNI registration tables and class-name strings are built at **runtime**
> (BSS + a separate decode), so the Java↔native method table itself is not
> statically recoverable; but the engine's subsystem strings are.

---

## 2. What the engine contains

### 2.1 VPN / TUN + a vendored userspace TCP/IP stack

```
com/android/vmcore/vpn/
set_socket_mark
mark: vpn net id is %d
/data/data/com.clone.android.dual.space/vm/vm%d/dev/tun
on_open_tun: recv failed (%d %s)
on_close_tun: close tun_fd %d from ...
bind_tun_add
init_tun_loop
add_vm_tun: epoll_ctl add vm tun failed (%d %s)
tun_loop
do_start_mgr: epoll_ctl add ctrl_fd failed (%d %s)
process_deferred_ops: send_ctrl_reply to %s failed (%d %s) for %s
/lib64/libnetd_client.so
init: dlsym android namespace failed (
```

It embeds a **userspace TCP/IP stack** — the string
`M_ROOMBEFORE(m) >= (sizeof(struct ip6) + sizeof(struct udphdr))` is an
**lwIP** macro, so `libvm.so` links **lwIP** to carry the guest's traffic over
the TUN device (rather than relying on the host's `netd`). The control channel
is `/data/data/.../vm/vm%d/dev/tun`; it uses `epoll`, `send_ctrl_reply`, and
`libnetd_client.so` for socket marking.

### 2.2 Netlink (virtual Wi-Fi/cellular)

```
netlink_client
/data/data/.../vm/vm%d/dev/netlink_server
Failed to send l2 packet bind reply to %s (%d %s)
Failed to send multicast reply to %s (%d %s)
Failed to get attribute: broken nl atrribute
Ignoring non netlink re[quest]
klog_vmcore_netlink
```
It bridges the host's netlink to the guest (`dev/netlink_server`).

### 2.3 Wi-Fi HAL

```
android/net/wifi/
wifi_channel: unsupported command %s
startScanWifi
_connectWifi
can not get JNIEnv for _connectWifi
netdevice-ioctl-v6
```

### 2.4 Virtual HALs (GPS / sensors / camera / phone)

```
com/android/vmcore/hal/
pipe:qemud:gps
_startGPS
_enableSensor is null
can not get JNIEnv for _enableSensor
_execPhoneCommand
on_camera_preview: cameraId is null
camera_channel: disconnect error %d
camera_channel: start error %d
new_channel
stopping hal service
```
The HAL forwarders use the **goldfish qemud pipe** (`pipe:qemud:gps`) and JNI
callbacks into the Java layer (`com.android.vmcore.hal.*`).

### 2.5 Input

```
/data/data/.../vm/vm%d/dev/input/touch
starting input service
input_write
%s: socket path too long (%s)
... SO_LINGER failed (%d %s)
```
The input service exposes `dev/input/touch` (the guest's EventHub connects to it;
see doc 13).

### 2.6 Audio

```
android/media/
on_audio_play: start
releaseAudioTrack: jni method is null
record: can not get ...
```

### 2.7 Display (GPU / framebuffer)

```
com/android/vmcore/hal/
on_hw_framebuffer_post
start display service gb failed (%d %s)
start display service gb2 failed (%d %s)
stopping display service
%s: setupSubWin...
/system/lib64/libui.so
init_libui: dlopen libui.so failed
init_libui
GraphicBuffer_Ctr
(JILandroid/view/Surface;IIF)Z          <- nativeAddSurface signature
```
The engine `dlopen`s the **host's** `/system/lib64/libui.so` (to construct
`GraphicBuffer`s) and manages sub-windows, plus a "gb"/"gb2" display service
(the GPU host-call path — see doc 19 §4.3).

### 2.8 Binder

```
get_binder_version
setup_binder: ftruncate binder file failed (%d %s)
setup_binder: mmap binder file failed (%d %s)
```
`setup_binder` (at `0x859f80`–`0x85a400`) calls **`ftruncate`** then
**`mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0)`** on a binder file
and parses the mapped region — the `VMBinderMgr` mechanism (doc 19 §4.4). The
engine's other functions call through **indirect function pointers** (`blr xN`
on vtable slots), i.e. JNI methods/callbacks, which is why a PLT-only call
extraction sees few calls.

### 2.9 Filesystem / process / misc

```
nativeFchmodat
nativeSetxattr
rm_rf: lstat %s failed (%d %s)
dev/vmproc/%d
%s/libk...
smaps_rollup
com/android/vmcore/VMNetworkConfig
(ILjava/lang/String;)Ljava/lang/String;   <- a JNI signature
Native registration unable to find class '%s'; aborting...
```

### 2.9b Linker namespace, HAL lifecycle, logging

The engine loads guest libraries through the **Android linker namespace API**:
```
android_dlopen_ext
android_get_LD_LIBRARY_PATH
android_create_namespace
init: dlsym android namespace failed (...)
```
i.e. it `dlsym`s `android_dlopen_ext`/`android_create_namespace` from the linker
and loads guest libs in a private namespace (the same mechanism the platform uses
for app namespaces).

It runs the virtual HALs as a **service** with a lifecycle
(`hal_service`, `stopping hal service`, `NATIVE_START_hal_MGR`), and logs each
subsystem to its own tag: **`klog_vmcore_gps`**, `klog_vmcore_netlink`,
`klog_vmcore_audio`, `klog_vmcore_render` — matching the Java `KLog`
(`klog_*` files). Netlink has its own loop (`bind: loop start`,
`stopping netlink service`); the camera channel parses focus commands
(`camera_channel: bad focus command %s`).

### 2.10 JNI surface (from recovered signatures + Java)
| Native method (Java) | Signature |
|---|---|
| `DisplayService.nativeAddSurface` | `(JILandroid/view/Surface;IIF)Z` |
| `VpiNativeUtils.decryptData` etc. | `(ILjava/lang/String;)Ljava/lang/String;` |
| `NativeUtil.*` (binder, fs, proc, tun, netlink, hal) | see doc 04 |

---

## 3. Engine architecture (synthesised)

```
libvm.so (the VM engine)
├── VPN/TUN + lwIP            (dev/tun, com/android/vmcore/vpn/*, libnetd_client)
├── Netlink bridge            (dev/netlink_server, klog_vmcore_netlink)
├── Wi-Fi HAL                 (android/net/wifi/*, startScanWifi, _connectWifi)
├── Virtual HALs              (com/android/vmcore/hal/*, pipe:qemud:gps, camera_channel,
│                              _startGPS, _enableSensor, _execPhoneCommand)
├── Input service             (dev/input/touch)
├── Audio                     (android/media/*, on_audio_play)
├── Display / GPU             (libui.so, GraphicBuffer_Ctr, gb/gb2, setupSubWin)
├── Binder (VMBinderMgr)      (setup_binder: ftruncate+mmap)
├── Process / fs              (vmproc/%d, nativeFchmodat, nativeSetxattr, rm_rf)
└── JNI registration          (runtime-built tables)
```

This matches exactly the Java layer (`NativeUtil`, `DisplayService`,
`InputService`, `AudioService`, `NetlinkManager`, `VpnManager`/`VpnTunService`,
`HALManager`, `BinderService`) — the Java declares the methods; `libvm.so`
implements them.

---

## 4. Reproduce

```bash
python3 workspace/notes/libvm_strings.py  workspace/notes/virtualmaster_libvm_strings.txt
python3 workspace/notes/libvm_strings2.py > workspace/notes/virtualmaster_libvm_strings_seg.txt
```

`libvm_strings.py` decodes one string per ciphertext run (540 strings);
`libvm_strings2.py` segments runs containing multiple strings (257 more). The
full lists are `workspace/notes/virtualmaster_libvm_strings*.txt`.
