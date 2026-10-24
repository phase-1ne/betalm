# device/lge/betalm — LineageOS 24.0 (G8s ThinQ)

Device-specific tree for the LG G8s ThinQ (codename **betalm**, model LM-G810EAW,
SoC Qualcomm SM8150 Snapdragon 855). It carries the product makefiles, the board
config and the device blobs list; almost everything else is in the shared
`device/lge/sm8150-common` tree it inherits.

- **Product:** `lineage_betalm` (`PRODUCT_DEVICE := betalm`, `PRODUCT_MODEL := G8s ThinQ`, `PRODUCT_MANUFACTURER := LGE`).
- **Lunch target:** `lineage_betalm-cp2a-userdebug` (release config `cp2a` = Android 16 QPR line).
- **Inherits:** `device/lge/betalm/device.mk` and `vendor/lineage/config/common_full_phone.mk`.
- **Kernel config consumed:** `TARGET_KERNEL_CONFIG := vendor/lineageos_beta_defconfig` (built from `kernel/lge/sm8150`).

## Upstream base

There is **no** official LineageOS 24.0 betalm branch. This tree forks from the
newest that exists, **`github/lineage-22.2`** (remote
`https://github.com/LineageOS/android_device_lge_betalm`).

- Fork point (merge-base): `ab686444ab6f849438c88eab2a5bc4b36c99b169` — "betalm: drop 32-bit camera stack" (2025-03-17).
- Local branch: `lineage-24.0-port` (local only; not pushed — the remote is the upstream LineageOS repo).

## Port commits (oldest → newest, `ab686444..lineage-24.0-port`)

| SHA | Subject | Why |
|---|---|---|
| `4481759` | betalm: decommonize audio HAL and DAC packages | Move the per-device audio HAL / Quad-DAC package wiring out of the common tree into betalm (`BoardConfig.mk`, `device.mk`). Commit body is only a `Change-Id`. |
| `cd95c0d` | betalm: Drop leftover LGE lights header | sm8150-common moved to the common Lineage lights AIDL service; the local `include/lge_lights.h` and the `LGE_LIGHTS_HAL` soong var are now unused. |
| `77fc4e2` | betalm: Use the hardware/lge libfpsph stub | Attempt: drop the stock `libfpsph.so` blob and use the hardware/lge stub (avoids a duplicate-module build failure). **Later reverted** — betalm does not use the LGE AIDL fingerprint stack. |
| `0dee180` | Revert "betalm: Use the hardware/lge libfpsph stub" | Branch tip. betalm runs `…biometrics.fingerprint-service.lineage` on the stock `fingerprint.msmnile.so`, so there is no duplicate module; ship the stock `libfpsph.so` again as 22.2 did. |

> During development there was one further local debug commit on top of `0dee180` that
> pre-authorized a developer adb key (an `adb_keys.debug` + `device.mk` lines), so a device
> that couldn't show the on-screen prompt could still be reached over adb in recovery / a
> hanging first boot. It was never meant for release and was **removed before publication**;
> no public build carries a pre-authorized adb key.

## How it fits the build

`breakfast betalm` / `lunch lineage_betalm-cp2a-userdebug` selects this product, which
inherits the common tree, pulls the betalm + sm8150-common vendor blobs, and builds the
`vendor/lineageos_beta_defconfig` kernel. Output: `m bacon` → an installable
`lineage-24.0-*-UNOFFICIAL-betalm.zip` (A/B, no dedicated recovery partition).

## Caveats

- A local debug adb-key commit existed during development and was removed before publication (see note above); no public build carries a pre-authorized adb key.
- `backup/pre-review` branch (`77fc4e2`) is a pre-review snapshot; not for build.
