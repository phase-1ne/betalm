# device/lge/sm8150-common — LineageOS 24.0

Shared device tree for LG SM8150 (Snapdragon 855) phones (G8/G8s/G8X/V50; betalm
inherits it). It holds the board config, the VINTF manifest, the HAL selection
(`sm8150.mk`), sepolicy pointers and the common blob fixups. Most of the 22.2→24.0
port work lives here.

## Upstream base

Forks from **`github/lineage-22.2`** (`android_device_lge_sm8150-common`); no official
24.0 branch exists.

- Fork point (merge-base): `49f6801466c1d911435b669001d72051a08ee74a` — "sm8150-common: Migrate to LGE AIDL LiveDisplay HAL" (2025-09-28).
- Local branch: `lineage-24.0-port` (local only).

## Kernel / boot image parameters (`BoardConfigCommon.mk`)

- `BOARD_BOOT_HEADER_VERSION := 2`, `BOARD_KERNEL_IMAGE_NAME := Image.gz`, `TARGET_KERNEL_SOURCE := kernel/lge/sm8150`, arm64, clang.
- Offsets: base `0x00000000`, kernel `0x8000`, ramdisk `0x01000000`, tags `0x00000100`, dtb `0x01f00000`; pagesize 4096. (These are the mkbootimg offsets; the abl kernel-window limit is a separate SoC constraint — see BUILD.md.)
- AVB enabled. The in-tree default AVB key is the standard AOSP test key (as for any unsigned/userdebug build); **public release builds are re-signed with the project's own release keys** (AVB key + releasekey) via the normal LineageOS signing flow, so signed OTA-style packages are possible. `BOARD_KERNEL_SEPARATED_DTBO := true`.

## Port commits (oldest → newest, `49f6801..lineage-24.0-port`)

Many are modelled on the oneplus sm8150-common / lge sm7250-common lineage-24.0 trees
(noted in each commit body).

| SHA | Subject | Why |
|---|---|---|
| `70ac49f` | sepolicy: move to hardware/lge | Move the common sepolicy into hardware/lge (large deletion here). Body is only a Change-Id. |
| `93c7002` | decommonize audio HAL and DAC packages | Counterpart to betalm `4481759`; strip audio/DAC package wiring from the common tree. Body is only a Change-Id. |
| `17628d9` | Use the hardware/lge device framework matrix | vendor/lineage dropped `config/device_framework_matrix.xml` on 23.x+; use the common LGE matrix from hardware/lge, drop the local copy. |
| `3269210` | Unset TARGET_CPU_ABI2 | Empty assignment no longer needed; newer build warns on it. |
| `8cd93cb` | Update some soong config variables to bool type | Soong config var type migration. |
| `5e1c356` | Opt out of speaker_layout_channel_mask field | ABI compat with audio HAL compiled before 15 QPR2 (LOS issue 9426). |
| `cabe430` | Use legacy libion implementation | 4.14 kernel only has legacy ION; several blobs link libion directly → build libion legacy + sepolicy. |
| `c433bd3` | Remove disable_configstore from PRODUCT_PACKAGES | Configstore removed from platform since Android 16 QPR2. |
| `6c98517` | fastbootd is now enabled by default | Platform adds fastbootd / `ro.fastbootd.available` for A/B itself. |
| `cd528fc` | Disable PRODUCT_OTA_ENFORCE_VINTF_KERNEL_REQUIREMENTS | FCM 7 wants a newer kernel than 4.14; the OTA-time kernel check can't pass. |
| `3654515` | Override kernel BPF version | 4.14 backports the eBPF features; report `android12-5.10` (`ro.bpf.kver_override`) so bpfloader accepts it. |
| `b9575d1` | Set FCM level to 7 | FCM levels < 7 no longer supported by the framework. |
| `bd33554` | power-libperfmgr: Update included soong namespaces | pixel moved pixelatoms deps behind namespaces; only power-libperfmgr needed. |
| `3eb685d` | Switch to the hardware/lge AIDL touch HAL | Use `vendor.lineage.touch-service.lge` (AIDL port of the local HIDL touch svc, same sysfs nodes); drop the local HIDL touch service + sepolicy. |
| `2731f8d` | Switch to the hardware/lge AIDL radio compat service | Use `android.hardware.radio-service.lge` (libradiocompat) + source-built lgdata@1.0. **Later reverted** (`1e525f6`) — relied on unmerged Gerrit. |
| `e2b5f03` | Switch to the LGE AIDL camera provider | Use hardware/lge AIDL camera provider. **Later reverted** (`a1dd09a`). |
| `7265b14` | Patch some blobs to depend on libtinyxml2-v34.so | libtinyxml2 10.1.0 (BP4A) is ABI-incompatible; redirect libdpps + the rescontrol powerhint common lib (32/64) to `libtinyxml2-v34.so`. |
| `0ed3bff` | Drop legacy ANT remnants | AntHalService app / `BOARD_ANT_WIRELESS_DEVICE` gone from QSSI; keep the vendor com.dsi.ant HAL blobs. |
| `f32c7c2` | Drop the explicit kernel FCM level from the manifest | A target-level-7 device manifest must not set `<kernel target-level>`; let the build derive it. |
| `fd8d7fa` | Switch to the QTI AIDL health HAL | HIDL health@2.1 not in any FCM≥7 matrix; use AIDL `android.hardware.health-service.qti` (+ recovery variant). |
| `1fbaf68` | Drop the generic context hub HAL | HIDL contexthub@1.0 not in any FCM≥7 matrix and no backing module ships; drop it. |
| `a1dd09a` | Revert "Switch to the LGE AIDL camera provider" | Keep the HIDL camera.provider@2.4 legacy provider for first boot (FCM 7 still accepts @2.4; oneplus ships it). |
| `1e525f6` | Revert "Switch to the hardware/lge AIDL radio compat service" | AIDL radio + source lgdata came from unmerged Gerrit (505682/3); go back to the HIDL wrapper + stock lgdata@1.0 prebuilt. |
| `63418f5` | Uprev the LGE radio HIDL wrapper to 1.5 | radio@1.4 not in FCM 7 (accepts 1.5–6); stock qcrild implements @1.5::IRadio; build `android.hardware.radio@1.5-service.lge`, declare @1.5. |
| `cc742ab` | **Declare the second-SIM LGE radio HAL instance** | HEAD. Add `@2.0::ILgeRadio/lge_radio2` to `manifest.xml`; without it the dual-SIM wrapper null-derefs on slot 2 → permanent OUT_OF_SERVICE (see FIXES.md). |

## How it fits the build

Inherited by `lineage_betalm`. `sm8150.mk` selects the HALs (radio HIDL wrapper,
camera.provider@2.4, health AIDL qti, legacy libion, touch AIDL); `manifest.xml`
declares the VINTF interfaces at FCM target-level 7; sepolicy now lives in hardware/lge.
