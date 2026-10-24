# Building LineageOS 24.0 for betalm (LG G8s ThinQ) — reproducible steps

Target: `lineage_betalm-cp2a-userdebug`. Built from a standard LineageOS 24.0 source
tree on an x86-64 Linux build host (a many-core workstation with enough RAM for a large
AOSP build), 2026-10-23. There is no official betalm 24.0 tree; the LG trees are the
`lineage-24.0-port` branches documented in the per-tree READMEs. Paths below use
`<AOSP_TOP>` for your checkout root — substitute your own.

## 1. Repo init + local manifest

```
repo init -u https://github.com/LineageOS/android.git -b lineage-24.0 \
    --git-lfs --depth=1 --no-clone-bundle
```

`.repo/local_manifests/betalm.xml` — LG trees pinned to `lineage-22.2` (newest that
exists) except hardware/lge which has a real `lineage-24.0` branch:

```xml
<manifest>
  <project name="LineageOS/android_device_lge_betalm"        path="device/lge/betalm"        remote="github" revision="lineage-22.2" />
  <project name="LineageOS/android_device_lge_sm8150-common" path="device/lge/sm8150-common" remote="github" revision="lineage-22.2" />
  <project name="LineageOS/android_kernel_lge_sm8150"        path="kernel/lge/sm8150"        remote="github" revision="lineage-22.2" />
  <project name="LineageOS/android_hardware_lge"             path="hardware/lge"             remote="github" revision="lineage-24.0" />
  <project name="TheMuppets/proprietary_vendor_lge_betalm"        path="vendor/lge/betalm"        remote="github" revision="lineage-22.2" clone-depth="1" />
  <project name="TheMuppets/proprietary_vendor_lge_sm8150-common" path="vendor/lge/sm8150-common" remote="github" revision="lineage-22.2" clone-depth="1" />
</manifest>
```

After sync, each LG tree is checked out to its `lineage-24.0-port` branch (the port
commits are local; see per-tree READMEs for base + SHAs).

## 2. Sync

```
git lfs install
repo sync -c -j16 --no-tags --no-clone-bundle --force-sync --force-checkout \
  || repo sync -c -j4 --no-tags --no-clone-bundle --force-sync --force-checkout
```

Measured: ~147 GB tree, first clean sync ~353 s with `--force-checkout`.
**Constraint:** never reuse a dir left half-synced from another branch — stale dirs not
in `project.list`, untracked cts files and half-done checkouts break Soong. Verify with
`repo forall -c 'git status --porcelain -uall'` before building.

## 3. Host env + build

```
export SKIP_ABI_CHECKS=true USE_CCACHE=1 CCACHE_EXEC=/usr/bin/ccache CCACHE_DIR=<AOSP_TOP>/.ccache
source build/envsetup.sh
lunch lineage_betalm-cp2a-userdebug
m -j46 bacon
```

- **`SKIP_ABI_CHECKS=true` is required** here: it drops `header-abi-dumper`,
  which otherwise dominated an incremental build. (`m bacon` output zip ≈ 1.27 GB.)
- Kernel build also needs `DTC_EXT` (external device-tree compiler) available.
- Put both the source tree **and** `out/` on SSD/NVMe. On a spinning HDD the build was
  I/O-bound (CPU 77% idle, iowait 57%): full `m bacon` took 9 h 23 min on HDD vs ~37 min
  with `out/` on SSD. `out/` as a symlink to an SSD subvolume was accepted without a full
  rebuild; source on a slow disk still costs ~7 min Kati/Soong re-analysis per config
  change.

### Staged validation (do NOT use full `m bacon` as first check)

VINTF errors only surfaced at 87–91% of a full build. Gate in order:

```
m -j46 check-vintf-all      # ~13 min; catches FCM/manifest errors early
m -j46 vendorimage
m -j46 bootimage            # 5–9 min; cheap kernel/abl iteration
m -j46 bacon                # full OTA zip
```

Build errors are under `out/siso_output`.

## 4. Critical constraints discovered

### 4a. Kernel image must fit the LG abl window (< 49.5 MiB)

The LG SM8150 abl loads the kernel at `0x80000` and rejects it if the arm64 `image_size`
(Image header u64 at offset 16) reaches the tags/DTB address `0x3200000` → **max
`0x3180000` = 49.5 MiB**. Over the window: `fastboot boot` says "Booting OKAY" but the
phone stays in fastboot and pstore is untouched.

Measured image_size: stock G810EAW40a `0x2e6c000` (46.4 MiB); post qcom-merge `0x35a6000`
(53.6 MiB, over). Defconfig (`vendor/lineageos_beta_defconfig`) changes to fit:

- Drop `IKCONFIG`/`IKHEADERS`/`KALLSYMS_ALL` → 49.65 MiB (still 152 KiB over).
- Drop r8152 USB-NIC + `DEBUG_LIST` + `SCHED_DEBUG` → ~0 further gain (49.64 MiB).
- **Disable `DEBUG_ALIGN_RODATA`** → segments 64 KiB-aligned instead of 2 MiB → **44.58
  MiB** (fits). `STRICT_KERNEL_RWX` retained for the RWX permission split.
- Then **re-enable `IKCONFIG` + `IKCONFIG_PROC`** (needed for `/proc/config.gz`, below) —
  ~100 KiB, still well inside 49.5 MiB.

### 4b. Device-tree nodes dropped by the merge

The qcom lineage-20 merge silently removed LG DT nodes (mem-offline, disp_rdump_region,
~434 /soc coresight/cti/etm/jtagmm nodes) from the sm8150 dtsi. The stock abl rejects the
resulting DTB. Fix: restore `arch/arm64/boot/dts` from lineage-22.2 (kernel commit
`71a2ba29e1f8`). **Rule:** after a kernel merge, diff `arch/arm64/boot/dts` against the
last-booting branch.

### 4c. Android 16 kernel-config requirement

Without `/proc/config.gz`, system_server aborts ("Kernel configs could not be fetched.
b/151092221") and the boot animation loops forever → `CONFIG_IKCONFIG_PROC` is mandatory
(kernel `1c5878e71e9e`).

### 4d. VINTF / FCM target-level 7

- `manifest.xml` set to FCM **target-level 7** (`b9575d1`); levels < 7 unsupported.
- A target-level-7 device manifest **must not** set `<kernel target-level>` (`f32c7c2`) —
  let the build derive it.
- `PRODUCT_OTA_ENFORCE_VINTF_KERNEL_REQUIREMENTS := false` (`cd528fc`) because 4.14 is
  older than FCM 7 wants, and `ro.bpf.kver_override` → `android12-5.10` (`3654515`) so the
  bpfloader accepts the backported eBPF.
- HAL fixes for FCM 7: HIDL health@2.1 → AIDL `health-service.qti` (`fd8d7fa`); drop
  contexthub@1.0 (`1fbaf68`); radio HIDL wrapper uprev @1.4 → @1.5 (`63418f5`); keep
  camera.provider@2.4 (`a1dd09a`). `CONFIG_HIDRAW`/`CONFIG_EROFS_FS` enabled in-kernel to
  satisfy FCM 7 (present via the kernel sync commit `e79b32851855` / merged vendor
  configs).

## 5. Flashing (summary — see the radio-diag notes and the lg-sm8150 unlock skill)

A/B device; no dedicated recovery. Boot image is the kernel + ramdisk; abl verifies the
image and the AVB signature. The public release is **signed with the project's own
release keys** (generated per the LineageOS signing-builds guide — AVB key + releasekey,
not the AOSP test keys), so signed OTA-style update packages can be produced. It remains
**UNOFFICIAL** with an **unlocked bootloader**, so Play Integrity will fail regardless of
signing. Builds have been deployed as a seamless A/B update via the Updater "Local
update"; a full OTA server is optional and not currently run.
