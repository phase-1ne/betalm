# FIXES — betalm 22.2 → 24.0 port changelog

Chronological (by dependency/discovery order) list of the non-trivial fixes, each with
its commit SHA, tree, and symptom → root cause → fix → verification. SHAs are from the
`lineage-24.0-port` branches of the LG trees (see the per-tree READMEs).

---

## 1. Kernel: merge qcom/lineage-20 instead of cherry-picking
- **Commit:** kernel `2d55f3a11ccf` (merge), parent2 `d05a02bda12f` (qcom/lineage-20 tip).
- **Symptom:** LG 4.14 kernel lacks CLONE3/pidfd/userfaultfd/EROFS/BPF backports that Android 13+ (24.0) userspace needs.
- **Root cause:** ~1586 individual commits would be needed to bring 22.2 LG kernel forward.
- **Fix:** merge `qcom/lineage-20` (`android_kernel_qcom_sm8150`) at the commit OnePlus used for its 24.0 merge (`d05a02bda12f`). 2756 files, +151784/−34724, 5 conflicts resolved.
- **Verified:** kernel compiles; boots after the follow-up fixes below.

## 2. Kernel: replace removed `__current_kernel_time()`
- **Commit:** kernel `e79e3177ea01`.
- **Symptom:** build break after the merge — 7 LG files call a helper the merge removed.
- **Root cause:** UPSTREAM `5beb6d289ab5` "timekeeping: Remove __current_kernel_time()" came in via the merge.
- **Fix:** switch the 7 LG touchscreen/power callers to `current_kernel_time()` (no NMI callers).
- **Verified:** compiles.

## 3. Kernel: sync generic defconfig options with sm8150-perf
- **Commit:** kernel `e79b32851855` (+ cmdline guard `29bb17abe1ac`).
- **Symptom:** 24.0 userspace needs features the 22.2 defconfig lacked.
- **Fix:** enable USERFAULTFD (ART uffd GC), EROFS + pcpu kthread, NET_CLS_MATCHALL / NET_ACT_POLICE / NET_ACT_BPF, CHACHA20POLY1305, WIREGUARD, DEFAULT_MMAP_MIN_ADDR=32768 across alpha/beta/flash/mh2 defconfigs.
- **Verified:** compiles; later needed by VINTF (HIDRAW/EROFS) and bpfloader.

## 4. Kernel: fit the LG abl kernel-size window (image_size saga)
- **Commits (in order):** `a77c8e4c1c88` → `60203660960d` → `27e1c70fd912` (→ later partially undone by `1c5878e71e9e`, fix #6).
- **Symptom:** `fastboot boot` of the new kernel returns "Booting OKAY" but the phone stays in fastboot; pstore untouched.
- **Root cause:** LG abl loads the kernel at `0x80000` and rejects it once `image_size` reaches the tags/DTB address `0x3200000` → max `0x3180000` = 49.5 MiB. Post-merge image_size was `0x35a6000` = 53.6 MiB (stock is 46.4).
- **Fix:** drop IKCONFIG/IKHEADERS/KALLSYMS_ALL (→49.65, still over); drop r8152 USB-NIC + DEBUG_LIST + SCHED_DEBUG (≈no gain); **disable DEBUG_ALIGN_RODATA** → 64 KiB section alignment instead of 2 MiB → **44.58 MiB** (fits). STRICT_KERNEL_RWX kept.
- **Verified:** boot-image bisect — our kernel under the window + correct DTB boots; measured image_size via the arm64 header u64 at offset 16.

## 5. Kernel: restore the lineage-22.2 LGE device trees
- **Commit:** kernel `71a2ba29e1f8`.
- **Symptom:** even an in-window kernel stays in fastboot (abl rejects the DTB).
- **Root cause:** the qcom merge silently dropped LG DT nodes (mem-offline, disp_rdump_region, ~434 coresight/cti/etm/jtagmm nodes) from the sm8150 dtsi; the stock abl refuses that DTB.
- **Fix:** restore `arch/arm64/boot/dts` (11 files) from lineage-22.2. Adding mem-offline alone (bisect variant D) was insufficient.
- **Verified:** boot-image bisect — same kernel with stock G810EAW40a DTB *or* the 22.2 DTB boots; with the merged DTB it does not.

## 6. Kernel: bring back IKCONFIG / IKCONFIG_PROC
- **Commit:** kernel `1c5878e71e9e` (HEAD).
- **Symptom:** first boot of the port loops on the boot animation forever.
- **Root cause:** Android 16 system_server aborts without `/proc/config.gz`: `android_os_Debug.cpp:801 Check failed: result == OK Kernel configs could not be fetched. b/151092221`.
- **Fix:** re-enable `IKCONFIG` + `IKCONFIG_PROC` (dropped in fix #4). ~100 KiB; image_size stays inside the abl window with IKHEADERS/KALLSYMS_ALL still off.
- **Verified:** boots to setup wizard.

## 7. VINTF / FCM target-level 7 (sm8150-common manifest + HAL selection)
- **Commits:** `b9575d1` (FCM 7), `f32c7c2` (drop explicit kernel level), `cd528fc` (disable OTA VINTF kernel check), `3654515` (BPF kver_override), `fd8d7fa` (health AIDL qti), `1fbaf68` (drop contexthub).
- **Symptom:** `check-vintf`/`assemble_vintf` errors at ~87–91% of a full build: health@2.1, contexthub@1.0 "in device manifest but not in framework compatibility matrix"; "Device manifest with target-level 7 must not explicitly set kernel level".
- **Root cause:** HIDL health@2.1 and contexthub@1.0 are not in any FCM≥7 matrix; FCM 7 forbids an explicit kernel level; 4.14 is older than FCM 7's kernel requirement.
- **Fix:** FCM level 7; switch health to AIDL `health-service.qti` (+recovery); drop contexthub (no backing module ships); remove `<kernel target-level>`; `PRODUCT_OTA_ENFORCE_VINTF_KERNEL_REQUIREMENTS := false`; `ro.bpf.kver_override` = android12-5.10.
- **Verified:** `m check-vintf-all` passes; boots.

## 8. Radio HAL: revert unmerged-Gerrit AIDL, use the HIDL wrapper @1.5
- **Commits:** sm8150-common `2731f8d` (AIDL, then) `1e525f6` (revert) → `63418f5` (uprev HIDL wrapper to 1.5); camera `e2b5f03` → `a1dd09a` (revert to HIDL @2.4).
- **Symptom / risk:** the AIDL radio compat service + source-built lgdata@1.0 depended on unmerged hardware/lge Gerrit changes (505682/505683); AIDL camera similarly unmerged.
- **Root cause:** building on unmerged Gerrit chains is fragile (third-opinion flagged the risk).
- **Fix:** revert to the merged-code path — HIDL radio wrapper, uprevved @1.4 → @1.5 (radio@1.4 not in FCM 7; stock qcrild implements @1.5::IRadio in libril-qc-hal-qmi.so), stock lgdata@1.0 prebuilt; keep HIDL camera.provider@2.4 (FCM 7 accepts @2.4).
- **Verified:** builds; modelled on oneplus sm8150-common / lge sm7250-common 24.0 trees.

## 9. Blob regen + libtinyxml2 ABI fixups
- **Commits:** vendor/betalm `2b2a497` + `7f0e35b`; vendor/sm8150-common `e35f29f` + `3dc0b90`; sm8150-common `7265b14` (extract-files patch).
- **Symptom:** libtinyxml2 10.1.0 (BP4A) is ABI-incompatible → prebuilts linked against the old lib crash; duplicate-module failures from AIDL-stack blobs.
- **Root cause:** libdpps + the LGE powerhint rescontrol common lib (32/64) NEEDED `libtinyxml2.so`; the AIDL switches changed which blobs ship.
- **Fix:** `extract-files.py <dump>` against a copy of the existing 22.2 blobs (nothing new downloaded), verified byte-identical by sha1/cmp except the 4 `libtinyxml2-v34` ELF fixups; ship the stock `libfpsph.so` (betalm) and `lgdata@1.0` (common) again after the AIDL reverts. Needs `PYTHONPATH` → `tools/extract-utils`.
- **Verified:** cmp/sha1 against the source blobs; builds.

## 10. Radio: dual-SIM wrapper SIGSEGV → OUT_OF_SERVICE (the headline fix)
- **Commits:** sm8150-common `cc742ab` (HEAD) + hardware/lge `35e4e2d` (HEAD).
- **Symptom:** both SIM slots permanently OUT_OF_SERVICE, baseband version empty, framework sees no radio — although qcrild runs, SIM cards are detected at the modem, and no SELinux denial.
- **Root cause (oracle, disassembly-confirmed):** the HIDL wrapper `android.hardware.radio@1.5-service.lge` does `ILgeRadio::getService("lge_radio2")` for slot 2; qcrild registers `@2.0::ILgeRadio/lge_radio2`, but that instance was **not declared** in `device/lge/sm8150-common/manifest.xml`. libhidl refuses lookup of undeclared HIDL instances → `getService` returns null → `Radio::setResponseFunctions` deref'd a null vtable at Radio.cpp:83 → the whole service SIGSEGV'd, killing both slots.
- **Fix:** (1) `cc742ab` — declare `@2.0::ILgeRadio/lge_radio2` in `manifest.xml`; (2) `35e4e2d` — null-guard `svc` in `Radio.cpp` so a missing LGE instance degrades (skips the proprietary hooks) instead of crashing.
- **Verified:** rebuilt (build16), deployed as a seamless A/B OTA. On slot a: `@1.5::IRadio/slot1`+`slot2` and `ILgeRadio/lge_radio`+`lge_radio2` registered, SIM1 **IN_SERVICE, registered on LTE**, baseband populated, 0 wrapper tombstones.
- **Reusable lesson:** a HIDL HAL that `getService()`es an instance will null-deref if that instance isn't in the VINTF manifest; declare every instance the HAL looks up (easy to miss the per-slot `lge_radio2` on multi-SIM).

## 11. sepolicy / build-cleanliness (hardware/lge + sm8150-common)
- **Commits:** hardware/lge `4dedf92` (ftm block dev), `b62271a` (SunlightEnhancement labels), `9457738` (dac_panel_app coredomain); sm8150-common `70ac49f` (move sepolicy to hardware/lge), plus the many "modelled on oneplus" cleanups (libion legacy `cabe430`, speaker_layout `5e1c356`, ANT `0ed3bff`, configstore `c433bd3`, fastbootd `6c98517`, framework matrix `17628d9`, etc.).
- **Symptom/cause/fix:** assorted FCM-7/Android-16 build and sepolicy requirements (e.g. "All platform apps must be labeled with a coredomain type" for QuadDACPanel). Each is a one-line config/label change; see README-sm8150-common.md / README-hardware-lge.md for the per-commit why.
- **Verified:** builds; boots to a working system.

---

### Development note (removed before publication)
- During development a local debug commit on device/betalm pre-authorized a developer adb key (an `adb_keys.debug` + `device.mk` lines) so a device that couldn't show the on-screen prompt could still be reached over adb in recovery / a hanging boot. This was never meant for release and was **removed before publication**; no public build carries a pre-authorized adb key.
