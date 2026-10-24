# Credits

This LineageOS 24.0 port for the **LG G8s ThinQ (betalm, LM-G810EAW)** stands entirely
on other people's work. The list below tries to name everyone whose code, trees, tools,
guides or documented dead-ends made it possible. It is deliberately generous and specific.

**If we missed you or got your handle wrong, tell us and we'll fix it — no ego here.**

Where a name comes from a git author record it is quoted as-is; where only a role is
known (e.g. a large upstream project), the role is described rather than a name invented.

---

## The ROM

- **The LineageOS project and its contributors** — for LineageOS itself, the device
  support infrastructure, and the LG trees this port forks from. We only ported it; we
  did not build the distribution.

## Upstream device / kernel / hardware trees (forked from)

This port forks the newest LineageOS LG trees that exist (`lineage-22.2`, and
`lineage-24.0` for `hardware/lge`) and carries local `lineage-24.0-port` branches on top.

- **device/lge/betalm** (base `ab686444`) and **device/lge/sm8150-common**
  (base `49f68014`): authored upstream by
  - **Han Sol Jin** (**lifehackerhansol**) — LG LineageOS device maintenance; the betalm
    and sm8150-common trees.
  - **Tuan Anh** — contributor to the sm8150-common tree.
- **kernel/lge/sm8150** (base `d4f335e13d73`): the LG SM8150 (msm-4.14) kernel, maintained
  within LineageOS on top of the Qualcomm / CodeAurora (CAF) base — see the SoC base below.
  Thousands of Linux kernel, Qualcomm and LineageOS contributors are in its history; we
  credit them collectively.
- **hardware/lge** (base `0ac6798`, a real upstream `lineage-24.0` branch): LGE HAL
  wrappers (radio, touch, camera, lights, fingerprint, lgdata) and common sepolicy.
  - **Han Sol Jin / lifehackerhansol** and **pnguyen879** — upstream authors (incl. the
    `@1.5` radio HIDL wrapper uprev the port's radio fix builds on).

## SoC kernel base

- **Qualcomm / CodeAurora Forum (CAF)** — the SM8150 (msm-4.14) kernel base the LG kernel
  derives from.
- **The Linux kernel community** — the underlying 4.14 kernel.
- **The LineageOS `android_kernel_qcom_sm8150` (`qcom/lineage-20`) tree** — merged into the
  LG 4.14 kernel at commit **`d05a02bda12f`** (the point OnePlus used for its 24.0 merge)
  to bring in the CLONE3 / pidfd / userfaultfd / EROFS / BPF backports Android 13+ needs.
- **The OnePlus SM8150 (`sm8150-common`) LineageOS 24.0 trees** — used as the reference for
  the merge point and for many HAL / sepolicy / build decisions during the port (health
  AIDL, camera provider choice, FCM-7 handling, libion legacy, etc.).

## Proprietary vendor blobs

- **TheMuppets** — for maintaining the proprietary LG vendor blob repositories
  (`proprietary_vendor_lge_betalm` / `proprietary_vendor_lge_sm8150-common`) the vendor
  side is regenerated from.

---

## Bootloader unlock, flashing & firmware tooling

The G8s unlock path (LG's official unlock server being dead) relies on work by:

- **Seasee606** and **Cloud1250000** — the LG SM8150 **engineering `abl`** that makes
  token-free `fastboot oem unlock` possible. (We do not redistribute the LG-signed image;
  get it from the original sources.)
- **cuynu** — the Linux **EDL unlock / flashing flow** this guide is built on.
- **netmsm** — the **LGUP crossflash** groundwork (common DLLs, device detect, UPGRADE mode).
- **bkerler** — **edlclient / `edl`** (the Qualcomm EDL client), without which none of the
  EDL backup, flash and unlock steps happen.
- **steadfasterX** — **SALT** (and the broader LG-on-Linux flashing/recovery knowledge),
  plus a maintained **kdztools** fork.
- **kdztools / KDZ handling** — **ehem** (original `kdztools`), **nuloperrito** (the C++
  `KDZTool` rewrite), and **iscgar** (KDZ parsing tooling). Used to inspect and extract
  stock KDZ firmware and to confirm Anti-Rollback = 0.
- **Qualcomm** — for the EDL / Firehose / Sahara protocols and the `android-tools`
  (fastboot/adb) the whole process depends on.
- Everyone in the LG G8 / G8s / G8X / V50 SM8150 forum threads who documented the dead ends,
  USB-ID behaviour, abl kernel-window limit and recovery quirks before us.

---

## AI assistance (full disclosure)

This was a first real port by a non-expert, and that is not hidden: the 22.2 → 24.0 port,
the boot and radio debugging, the kernel-size work, and the builds were done with **heavy
AI assistance** — primarily **Claude (Anthropic)**, with **GPT / Codex (OpenAI)** also
assisting. The AI helped read disassembly, reason about the VINTF/FCM and HAL failures, and
draft the trees and docs; a human drove it, tested on real hardware, and is responsible for
the result. Mistakes are ours, not the upstream projects'.

---

## This port

- **The phase-1ne maintainer** — this port, the boot/radio debugging, the build and these
  docs. *[MAINTAINER: add your XDA / GitHub handle here for public credit.]*
- **Everyone who tests, files logs or sends fixes** — you are doing the maintaining a
  single-device one-off port can't promise on its own. Thank you.

---

*Missing from this list? Open an issue / reply on the thread with your name or handle and
what you worked on, and we'll add you.*
