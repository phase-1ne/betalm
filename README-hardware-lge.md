# hardware/lge — LineageOS 24.0 (port additions)

Shared LGE hardware tree: HIDL/AIDL HAL wrappers (radio, touch, camera, lights,
fingerprint libfpsph stub, lgdata) and common sepolicy for all LGE devices. Unlike the
device/kernel trees, hardware/lge **has a real upstream `lineage-24.0` branch**, so the
port only adds a few commits on top of it.

## Upstream base

- Remote `github/lineage-24.0` (`android_hardware_lge`), tip `0ac6798` "hidl: radio: Uprev to 1.5" (2026-09-26) — this is also the fork point (merge-base).
- Local branch: `lineage-24.0-port` (local only).

## Port commits (`github/lineage-24.0..HEAD`, newest first)

| SHA | Subject | Why |
|---|---|---|
| `35e4e2d` | **hidl: radio: Do not crash when an LGE radio instance is missing** | HEAD. `Radio::setResponseFunctions` called `svc->setResponseFunctions()` with `svc = ILgeRadio::getService()` unchecked; on a dual-SIM device missing `lge_radio2` it returns null → SIGSEGV kills the whole radio service (both slots). Null-guard so a missing LGE instance degrades. Our fix. |
| `4dedf92` | sepolicy: Add ftm block device and its permissions | Cherry-pick (upstream LineageOS/issues #245): label the `ftm` block device + `mdm_helper` perms. |
| `b62271a` | sepolicy: label SunlightEnhancement nodes | genfs labels for the SunlightEnhancement sysfs nodes. |
| `9457738` | sepolicy: private: Mark dac_panel_app as coredomain | QuadDACPanel is a platform app and must carry a coredomain type, else "All platform apps must be labeled with a coredomain type" fails the build. |

`35e4e2d` is the port's own work (co-authored, no Change-Id). `4dedf92`/`b62271a`/
`9457738` carry upstream Change-Ids / cherry-pick trailers — generic LGE fixes applied so
betalm builds and boots cleanly on 24.0.

## How it fits the build

sm8150-common consumes hardware/lge throughout the port: the device framework matrix
(`17628d9`), the sepolicy (`70ac49f` moved it here), the AIDL touch HAL (`3eb685d`), the
libfpsph stub (betalm `77fc4e2`, then reverted), and the `@1.5` LGE radio HIDL wrapper
(`android.hardware.radio@1.5-service.lge`, upstream `0ac6798`) that the port's radio fix
(`35e4e2d`) hardens.
