# STATUS — LineageOS 24.0 for betalm (LG G8s ThinQ)

Build: `lineage-24.0-20261023-UNOFFICIAL-betalm`, kernel `4.14.357-openela-perf+`,
Android release 17 (16 QPR line, `cp2a`). First-boot hardware sweep 2026-10-24 (adb
dumpsys/lshal/getevent); radio verified after the build16 radio fix. Device boots to the
setup wizard and runs.

## Works (verified)

| Component | Evidence |
|---|---|
| **Touchscreen** | `touch_dev` 1080×2248; setup wizard driven by hand. |
| **WiFi** | associated, DHCP lease obtained, 11ac link up. |
| **Camera** | 10 camera devices, provider HAL up; **owner confirmed it actually captures (2026-10-24).** |
| **Audio** | `audioserver` + `android.hardware.audio.service` running. |
| **Sensors** | accel, gyro, magnetometer, light, pressure, proximity, gravity all enumerated. |
| **Fingerprint** | `FingerprintProvider` (`…fingerprint-service.lineage` on stock `fingerprint.msmnile.so`); **owner confirmed enroll + unlock (2026-10-24).** |
| **Cellular** | After the dual-SIM radio fix (sm8150-common `cc742ab` + hardware/lge `35e4e2d`), slot a: `@1.5::IRadio/slot1`+`slot2` registered, **SIM1 IN_SERVICE, registered on LTE**, baseband populated, 0 wrapper tombstones. |
| Bluetooth stack | present; SoC version read at boot (stack up; pairing/audio not exercised). |

## Untested / not confirmed

- **GPS** fix
- **NFC**
- **Widevine / DRM level**, video playback DRM
- **Quad-DAC** audio (hardware present: QuadDACPanel app labelled; function not exercised)
- **VoLTE** / IMS, in-call audio
- **SIM2 / dual-SIM cellular in service** — slot2 HAL instance now registers, but a second SIM was not confirmed IN_SERVICE
- Calls / SMS / mobile data throughput beyond "IN_SERVICE"
- Charging / thermals over time
- Bluetooth pairing and A2DP

## Known caveats

- **UNOFFICIAL** build, signed with the **project's own release keys** (AVB key +
  releasekey, not the AOSP test keys), so signed OTA-style packages are possible; no OTA
  server is run yet, so updates are manual "Local update" / flash. Because it is
  unofficial with an **unlocked bootloader**, Play Integrity fails regardless of signing.
- No official LineageOS betalm 24.0 tree — the device/kernel trees are the
  **`lineage-24.0-port` branches** of the LG trees. hardware/lge has a real upstream 24.0
  branch + 4 local commits.
- Kernel is 4.14 with FCM-7 enforcement relaxed
  (`PRODUCT_OTA_ENFORCE_VINTF_KERNEL_REQUIREMENTS := false`, `ro.bpf.kver_override`).
- A local debug commit (pre-authorizing a developer adb key) existed on device/betalm
  during development and was **removed before publication**; no public build carries a
  pre-authorized adb key.
- **Maintenance uncertain:** single-device, one-off port; no CI, no release channel.

## Discrepancy note (transcript vs git — see report)

The first-boot hwtest sweep recorded "the installable zip predates the IKCONFIG fix; the
phone runs a separately-flashed boot image; a clean zip rebuild is pending." The IKCONFIG
fix **is** committed (kernel `1c5878e71e9e`) and the later radio fix (build16) was
deployed as an A/B OTA with cellular verified. So source-wise the tree is complete; what
is **not** verifiable from git is whether a single clean `m bacon` zip containing both the
IKCONFIG and radio fixes was produced and archived (git holds source, not build artifacts).
