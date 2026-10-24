# kernel/lge/sm8150 — LineageOS 24.0 port

Qualcomm SM8150 kernel (msm-4.14) for LG G8/G8s/G8X/V50. The betalm port builds
defconfig **`arch/arm64/configs/vendor/lineageos_beta_defconfig`** (arm64, clang,
`Image.gz`).

## Upstream base and the merge model

No official LineageOS 24.0 kernel branch exists, so the port **merges** the newer QCOM
LineageOS kernel into the 22.2 LG kernel rather than cherry-picking:

- 22.2 base (merge-base with `github/lineage-22.2`): `d4f335e13d73` — "techpack: audio: tfa98xx: fix another -Wenum-compare warning" (2025-03-15).
- Merged-in branch: **`qcom/lineage-20`** (`android_kernel_qcom_sm8150`), at the commit OnePlus used for its 24.0 merge: **`d05a02bda12f`** "UPSTREAM: arm64: Implement copy_thread_tls" (2026-09-09).
- Merge commit: **`2d55f3a11ccf`** "Merge remote-tracking branch 'qcom/lineage-20' into lineage-24.0-port" — parents `d4f335e13d73` (LG 22.2) + `d05a02bda12f` (qcom). 2756 files changed, +151784 / −34724; the session noted 5 conflicts resolved.
- Local branch: `lineage-24.0-port` (local only).

Rationale: the LG 4.14 kernel needed the CLONE3 / pidfd / userfaultfd / EROFS / BPF
backports that the qcom lineage-20 tree already carries for Android 13+ userspace;
merging brings them in as one coherent tree instead of ~1586 individual cherry-picks.

## Port commits on top of the merge (`merge → HEAD`, newest first)

| SHA | Subject | Why |
|---|---|---|
| `1c5878e71e9e` | **defconfig: Bring back IKCONFIG and IKCONFIG_PROC** | HEAD. Android 16 system_server aborts without `/proc/config.gz` ("Kernel configs could not be fetched. b/151092221") → boot-anim loop. Re-enable; ~100 KiB, still inside the abl window. |
| `71a2ba29e1f8` | **arm64: dts: Restore the lineage-22.2 LGE device trees** | The qcom merge dropped mem-offline / disp_rdump_region / coresight·jtagmm nodes from the sm8150 dtsi; the LG abl then rejects the boot image → fastboot. Restore `arch/arm64/boot/dts` from 22.2 (11 files). |
| `27e1c70fd912` | defconfig: Disable DEBUG_ALIGN_RODATA to fit the abl kernel window | RODATA 2 MiB section alignment made image_size shrink only in 2 MiB steps. Off → 64 KiB alignment; `STRICT_KERNEL_RWX` keeps the RWX split. Biggest size win (→ 44.58 MiB). |
| `60203660960d` | defconfig: Drop the Realtek USB NIC driver, DEBUG_LIST and SCHED_DEBUG | r8152/r8156 carries ~210 KiB firmware/code; DEBUG_LIST/SCHED_DEBUG are debug aids. Shaved toward the window before the RODATA fix. |
| `a77c8e4c1c88` | defconfig: Drop IKCONFIG, IKHEADERS and KALLSYMS_ALL to fit the abl kernel window | First size-trim attempt: post-merge image_size 53.6 MiB vs stock 46.4 vs window 49.5. (IKCONFIG later restored by `1c5878e` once the window was met via RODATA.) |
| `e79e3177ea01` | treewide: lge: Replace removed __current_kernel_time() | UPSTREAM `5beb6d289ab5` (from qcom merge) removed the helper; 7 LG touchscreen/power files use `current_kernel_time()` instead (no NMI callers). |
| `e79b32851855` | ARM64: configs: alpha/beta/flash/mh2: Sync generic options with sm8150-perf | Enable USERFAULTFD (ART uffd GC), EROFS(_FS_PCPU_KTHREAD), NET_CLS_MATCHALL / NET_ACT_POLICE / NET_ACT_BPF, CHACHA20POLY1305, WIREGUARD, DEFAULT_MMAP_MIN_ADDR=32768 across all four variant defconfigs. |
| `29bb17abe1ac` | fs: proc: cmdline: Guard offset_addr and proc_command_line_tail | Backport fixing `74354d91fcf5` (PROC_CMDLINE_APPEND_ANDROID_FORCE_NORMAL_BOOT). Cherry-pick `4b3045d8076e`. |

Everything below `2d55f3a11ccf` in `git log` comes in via the qcom/lineage-20 merge and
is upstream history, not port work.

## The abl kernel-size constraint (SM8150-specific)

The LG SM8150 abl loads the kernel at `0x80000` and refuses it if
`image_size` (arm64 Image header u64 at offset 16) reaches the DTB/tags load address
`0x3200000` → **max `0x3180000` = 49.5 MiB**. Over the window, `fastboot boot` reports
"Booting OKAY" but the phone stays in fastboot and pstore is untouched. Measured:
stock G810EAW40a `0x2e6c000` (46.4 MiB); post-merge `0x35a6000` (53.6 MiB); after the
defconfig trims + DEBUG_ALIGN_RODATA off: **44.58 MiB** (fits). See BUILD.md.

## How it fits the build

`device/lge/sm8150-common` sets `TARGET_KERNEL_SOURCE := kernel/lge/sm8150` and
`device/lge/betalm` sets `TARGET_KERNEL_CONFIG := vendor/lineageos_beta_defconfig`.
The kernel builds in-tree during `m bacon` / `m bootimage` (needs `DTC_EXT` and
`SKIP_ABI_CHECKS=true` in the host env — BUILD.md).

## Other branches

- `test/22.2-hosttools` (`f2ceafb85414`) — experiment taking sign-file/extract-cert host
  tools from the 24.0 port because BoringSSL has no OpenSSL ENGINE; confirms a stock 22.2
  kernel does **not** build in the 24.0 tree.
