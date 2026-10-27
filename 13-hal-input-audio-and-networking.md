# 13 — HAL, Input, Audio & Networking (deep dive)

Everything the guest would get from hardware — touch, sensors, camera, audio,
Wi-Fi, GPS — must be synthesized by the host app. This doc covers the virtual
HAL layer and the I/O paths.

---

## 1. The HAL problem

A normal Android guest talks to hardware through HALs (HIDL/AIDL services that
load `<board>` implementations from `/vendor/lib64/hw/*.so`). A container has no
hardware, so it must either:

* **reuse the emulator's goldfish HALs** (which already virtualize hardware),
  or
* **provide custom HAL implementations** that source data from the host or from
  synthetic values.

The guest's `ro.hardware` / `ro.hardware.<x>` properties select which
implementation loads.

| HAL | twoyi | Virtual Master | VMOS Pro |
|-----|-------|------|----------|
| composer | `hwcomposer.twoyi.so` (custom) | in `libvm.so` | in `libvmtools.so` |
| gralloc | `gralloc.goldfish.so` | ranchu gralloc | ranchu gralloc + `NATIVEBUFFERsocket` |
| camera | goldfish | native HAL service | `libhalbox.so` (`CameraPreview/Picture/OnStop`) |
| GPS/GNSS | goldfish | native HAL service | `libhalbox.so` (`GpsChanged`, `GpsSVInfoList`, `GnsSVInfoList`) |
| Wi-Fi | goldfish | `NetlinkManager` | `libhalbox.so` (`WIFIChanged`) |
| sensors | goldfish | native HAL service | `libhalbox.so` (`SensorChanged`), `Hal51Tools.sendSensorData` |
| GSM/telephony | goldfish | native HAL service | `libhalbox.so` (`initGSMData`) |
| audio | goldfish | `AudioService` | `libnative-lib.so` (`NativeAudioManagerUtil`) |
| battery/rtc | goldfish | goldfish | goldfish |
| vibrator/power/keymaster/gatekeeper | goldfish | goldfish | goldfish |

Evidence:
* twoyi: `system/vendor/lib64/hw/*` contains `gralloc.goldfish.so`,
  `hwcomposer.twoyi.so`, `audio.primary.default.so`, `power.default.so`,
  `vibrator.default.so`, `android.hardware.*@*-impl.so`; `ro.hardware=goldfish`,
  `ro.hardware.hwcomposer=twoyi`.
* VMOS Pro: `libhalbox.so` exports the JNI in the table above;
  `Hal51Tools.{sendSensorData,sendWifiData}`; `SensorManagerUtil.SensorChanged`.
* Virtual Master: `com.android.vmcore.hal.{DisplayService,AudioService,NetlinkManager}`,
  `com.android.vmcore.vpn.{VpnManager,VpnTunService}`.

---

## 2. Input

Input is the clearest "device server" pattern.

### 2.1 The normal path

Host kernel → `EventHub` reads `/dev/input/event*` → `InputReader` →
`InputDispatcher` → app. Events are `struct input_event { type, code, value,
time }` (evdev), with multitouch via `EV_ABS`/`ABS_MT_*`.

### 2.2 twoyi (explicit and readable)

`app/rs/src/input.rs` runs two servers, each presenting a virtual input device:

* `<rootfs>/dev/input/touch` (device `vtouch`) and
  `<rootfs>/dev/input/key0` (device `vkey`).
* Protocol: the guest connects; the server writes a **`struct device_info`**
  (name, driver_version, `input_id`, physical_location, unique_id, key/abs/rel/
  sw/led/ff/prop bitmasks, abs_min/max) — i.e. exactly what `EVIOCG*` would
  return — then streams **`struct input_event`** records.
* The host `MotionEvent` is converted to multitouch:
  `ABS_MT_SLOT`, `ABS_MT_TRACKING_ID`, `ABS_MT_POSITION_X/Y`, `ABS_MT_PRESSURE`,
  `BTN_TOUCH`/`BTN_TOOL_FINGER`, `SYN_REPORT`; up/cancel send
  `ABS_MT_TRACKING_ID = -1`.
* `sendKeycode` emits `KEY_BACK`.

So the guest's input HAL reads these sockets instead of real evdev devices.

**Guest side:** the guest's `libinputflinger.so` `EventHub` is patched —
`openDeviceLocked(path)` does `socket(AF_UNIX,SOCK_STREAM,0)` + `connect(path)`
instead of `open()`ing a char device, then reads the device descriptor. The
`connect` runs through the patched libc `translate_socket`, so
`/dev/input/touch` resolves to `<rootfs>/dev/input/touch`. `libinput.so` (the
framework side) is stock; the customization is entirely in `libinputflinger.so`.

### 2.3 Virtual Master / VMOS Pro

* Virtual Master: input delivered at `<vmDir>/dev/input/touch`.
* VMOS Pro: `rftouchNNN` socket + JNI `vmtools_input_event64(vmId, action,
  MotionEvent.nativePtr, pressure)` / `vmtools_input_keyevent64(keyCode)`;
  wrappers extract `MotionEvent.mNativePtr` by reflection.

---

## 3. Audio

### 3.1 twoyi

Uses the emulator's goldfish audio HAL (`audio.primary.default.so`,
`libaudiohal.so`, `android.hardware.audio@2.0-impl.so`). No custom audio bridge
in the app layer.

### 3.2 Virtual Master / VMOS Pro

* Virtual Master: `AudioService` (native) in `libvm.so`.
* VMOS Pro: `NativeAudioManagerUtil`:
  * **Guest→host playback**: `startAudioPlayer("<vmDir>/rfaudio", 44100)`;
    decoded PCM arrives via `AudioWriteData(byte[],off,len)` → host `AudioTrack`.
  * **Host→guest mic**: either `AudioRecord` + SoundTouch → native
    `recordingWrite(...)`, or a `LocalServerSocket` (`SocketAudioRecorder`)
    feeding 1280-byte frames on a 200 ms cadence.
* `libmci.so` carries audio/video packets (`InputAudioPack`, `InputVideoPack`,
  `AvAudio_pack`, `AvVideo_pack`) with AAC/H.264 via FFmpeg — used for the
  remote/streaming path.

---

## 4. Networking

### 4.1 Virtual Master (readable)

* `VMVpnService` extends `android.net.VpnService`; `bindProcessToNetwork`.
* `VpnTunService` natives:
  `nativeInitTunLoop/nativeStartTunLoop/nativeAddVMTun/nativeDelVMTun/
  nativeStopTunLoop` — a native TUN loop that carries the guest's traffic.
* `NetlinkManager` reads the host's Wi-Fi/cellular netlink and pushes
  `VMNetworkConfig` into the guest:
  `ifname, mac, ip, network, netmask, gateway_ip, ipv6_prefix, dns4_ip,
  dns6_ip`.

### 4.2 VMOS Pro

* `KeepLiveVpnService` (VpnService) for keep-alive.
* `NativeUtil.start_pipe` / `vknel_pipe` for the kernel channel; netlink via the
  same `NetlinkManager` design.

### 4.3 twoyi

* Uses the emulator's `init.goldfish.rc` network props (`net.eth0.*`,
  `net.dns1`) and the guest's own `netd`. No host VPN bridge.

---

## 5. Properties, log and time

* **Property service**: a UNIX socket (`<rootfs>/dev/socket/property_service`
  for twoyi). The guest's `libc`/`init` read/write properties there; the host
  can seed `default.prop`/`vendor/default.prop`.
* **Log**: guest `logd` sockets are redirected; twoyi's `BootLogTexture` streams
  the guest's `logcat -v brief *I` into a `TextureView` during boot.
* **Time/locale**: the host seeds `persist.sys.language/country/timezone` and
  `ro.sf.lcd_density` (twoyi `RomManager.initRootfs`); Virtual Master uses
  `SET_LOCALE_AND_TIME` over the event bus.

---

## 6. Building your HAL box (clean-room)

1. Start from AOSP's **goldfish HALs** (Apache-2.0) — they already virtualize
   camera/GPS/sensors/audio/battery for the emulator.
2. Add a **camera** implementation that feeds a real host camera frame (or a
   synthetic test pattern).
3. Add **GNSS/Wi-Fi/sensors** implementations fed by the host:
   `SensorManager`, `LocationManager`, `WifiManager` (mirroring Virtual Master's
   `NetlinkManager`/`Hal51Tools`).
4. Keep a **synthetic fallback** for every HAL so the guest boots even when the
   host denies a permission.
5. Implement input as a **device server** (twoyi's `device_info` + `input_event`
   protocol is the reference).
6. Implement audio as a **PCM bridge** (guest HAL ↔ socket ↔ `AudioTrack`/
   `AudioRecord`).
7. Network via **VpnService + TUN** (Virtual Master's model) rather than trying to
   reproduce `netd`.
