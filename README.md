# LineageOS 24.0 for the LG G8s ThinQ (betalm) — unofficial

An **unofficial, experimental** build of LineageOS 24.0 (Android 17) for the LG G8s ThinQ (LM-G810, codename `betalm`, Snapdragon SM8150). There is no official LineageOS for this device past 22.2 and no maintainer — this is a community effort to get 24.0 running and share it so others can test it, build on it, or take it over.

**Done, honestly, with heavy AI assistance** by someone who isn't an experienced ROM developer. We're sharing it as-is and will try to maintain it as time allows, but can't promise to. The device trees are public so anyone can continue the work. See [CREDITS.md](CREDITS.md) — a lot of other people's work made this possible.

## Status

**Working** (verified on-device): boot, touchscreen, Wi-Fi, camera, audio, sensors, fingerprint (enroll + unlock), **cellular/LTE** (voice + data register).
**Not yet tested:** GPS, NFC, Widevine/DRM level, Quad-DAC, VoLTE, second SIM.
**Caveats:** unofficial; **release-key signed with our own keys** (not AOSP test keys); the bootloader must be unlocked, so Google **Play Integrity will fail** (banking/attested apps won't work). No official OTA server.

Full detail: [STATUS.md](STATUS.md).

## Download

Get the latest zip from the [**Releases**](../../releases) page: `lineage-24.0-20261024-UNOFFICIAL-betalm.zip`.

## Install

Unlock the bootloader, boot LineageOS recovery, factory reset, sideload the copy-partitions zip, then the ROM. Full step-by-step (including the LG G8s bootloader unlock): see the XDA guide *(thread link added after posting)*.

## Source

| Tree | Repo |
|---|---|
| device | [android_device_lge_betalm](https://github.com/phase-1ne/android_device_lge_betalm/tree/lineage-24.0) |
| common device | [android_device_lge_sm8150-common](https://github.com/phase-1ne/android_device_lge_sm8150-common/tree/lineage-24.0) |
| kernel | [android_kernel_lge_sm8150](https://github.com/phase-1ne/android_kernel_lge_sm8150/tree/lineage-24.0) |
| hardware | [android_hardware_lge](https://github.com/phase-1ne/android_hardware_lge/tree/lineage-24.0) |
| vendor | [proprietary_vendor_lge_betalm](https://github.com/phase-1ne/proprietary_vendor_lge_betalm/tree/lineage-24.0) |
| common vendor | [proprietary_vendor_lge_sm8150-common](https://github.com/phase-1ne/proprietary_vendor_lge_sm8150-common/tree/lineage-24.0) |

Build instructions: [BUILD.md](BUILD.md). What was fixed to make it boot: [FIXES.md](FIXES.md).

## Disclaimer

Flashing custom software can brick your device, and will void your warranty. We are not responsible for anything that happens to your phone. Make full backups first.
