# Linux kernel for the Huawei MediaPad T2 8.0 Pro (JDN-L01, JDN-W09, JDN-AL00)

Kernel 3.10.108 for the Huawei MediaPad T2 8.0 Pro / Honor Pad 2
(codename `jordan`, MSM8939v2). Builds **TWRP 3.7.0** and
**LineageOS 19.1 (Android 12.1)**.

When this port started, no kernel source for this device was available, so
the tree is based on the **kiwi** (Honor 5X) kernel, which shares the SoC:

- Base: https://github.com/mohammadsa9/android_kernel_huawei_kiwi (`lineage-18.1`)
- Which is in turn derived from Huawei's released kiwi kernel source.

All credit for the base tree goes to **@mohammadsa9**, whose kiwi work
made this port possible.

Huawei's own kernel source for this tablet (the `JDN-W09_MM_EMUI4.0`
open-source package) surfaced later, shared by a JDN-W09 owner. The
`jdn-pad-power` branch ports what this tree was missing from it — see
Phase 3.

This repository was called `android_kernel_huawei_jdnl01` until it gained
the W09 and AL00; GitHub redirects the old URL.

## Branches

| Branch | Contents |
| --- | --- |
| `jdn-pad-power` | `lineage-19.1` plus Phase 3 (Huawei JDN source port, JDN-W09 and JDN-AL00 DTBs). Under test. |
| `lineage-19.1` | The LineageOS 19.1 release: Phases 1 and 2. |
| `lineage-18.1` | The original fork point, kept so the relationship to the upstream kiwi tree stays visible. Phase 1 (TWRP) only. |

The `lineage-18.1` name is inherited from the kiwi branch this was forked
from; it does not mean the kernel is limited to LineageOS 18.1.

## Phase 1 — bring-up (TWRP)

| Change | Why |
| --- | --- |
| Board DTS + defconfig | Decompiled from the stock RECOVERY.img DTB. Two panel-supply entries with empty `qcom,supply-name` had to go: `of_property_read_string` returns `-EILSEQ` on a zero-length property, which aborts the DSI0 probe and leaves the device with no framebuffer. |
| Novatek touch driver | This unit ships a Novatek digitiser. Its chip ID cannot be read, but it runs its own firmware and reports in the standard Novatek point format, so the driver just powers it, resets it and decodes points. |
| MDSS backlight GPIOs | The backlight sits on three GPIOs no CAF property name covers, so nothing drove them and the brightness node lit nothing. |
| gcc-wrapper python 3 | Builds on a modern host. |

## Phase 2 — LineageOS 19.1

| Change | Why |
| --- | --- |
| **LM36923 backlight driver** (`drivers/video/lm36923.c`) | The panel node declares `bl_ctrl_dcs`, but this panel ignores DCS `0x51`: Huawei's own kernel replaced `mdss_dsi_panel_bklt_dcs()` with a `_pad` variant driving a TI LM36923 at i2c-0 `0x36`. Register map recovered from the stock kernel's strings, init order from aboot, then verified against the chip over i2c before any code was written. Register `0x19` holds brightness bits 10:3 and maps 1:1 onto `bl_level`. |
| **AP3426 light + proximity** | The DTS declares an APDS-9930 and a BH1745 that are not fitted; the real part is a Dyna Image AP34xx at `0x1e`. Adding `"di,ap3426"` as a second compatible binds the in-tree driver unmodified. Tuning values come from the identically wired ql790 reference. |
| **bma2x2 12-bit fix** | `BMA2X2_SENSOR_IDENTIFICATION_ENABLE` compiled out the `>> (16 - bitwidth)` normalisation, so the 12-bit BMA255 emitted raw 16-bit values. The HAL scaled them by the declared resolution and produced ~157 m/s², which the rotation judge rejected as external acceleration. |
| **rmtfs sharedmem address** | A previous commit moved the rmtfs UIO window off Huawei's `0x86700000` to avoid a 384 KB overlap with `modem_adsp_mem`. That overlap is real, but it is also what Huawei's shipping DTB does. At the moved address, a working `rmt_storage` mapping uio0 with the modem live takes an unrecoverable exception (`Bad mode in Error handler`) and the system dies in milliseconds. Restored, and the modem subsystem enabled. |
| **Touch gated on display state** | The controller emits a spurious tap ~250 ms after the power key while the panel powers down, which woke the screen straight back up. Reporting stops on blank and resumes after unblank. |

## Phase 3 — Huawei's JDN source (`jdn-pad-power`)

Every source file this kernel builds was compared with Huawei's copy
(Huawei's `camera_v2` is our `camera_v2_kiwi`), counting Huawei's
`#ifdef`-guarded blocks and ticket comments. The differences that matter on
this hardware were ported, most of them measured on a JDN-L01 first.

| Change | Why |
| --- | --- |
| **Pad panel power sequence** (mdss) | Huawei drives the NT51021 pad panels through a tablet-specific path: reset never pulsed after the bootloader, VLED/BL/VCC ordered with Huawei's delays, brightness through Huawei's `jordan` curve. Pulsing reset with VCC up made the image wake up shifted a few rows (reported on a W09, reproduced on the L01). With a real VCC power cycle the LM36923's PWM input stays at zero, so its control register is set to register-only mode (`0x11` = `0x15`). Verified: 6/6 wakes lit, no shift. |
| **DSI command-injection sysfs removed** | `/sys/kernel/jdn_dsi/cmd` was a bring-up tool for the backlight work that shipped in `lineage-19.1`; a root-writable arbitrary DSI command interface has no place in a release. |
| **Wall-charger input limit 2000 mA** | Stock JDN ships `ti,max_cin_limit_currentmA = 1000`. With Huawei's 5 V / 2 A adapter the BQ24296 sat DPM-limited at 1 A and the battery got about 0.4 A with the screen on. Every Honor 5X board with the same charger and driver uses 2000. Temperature limits and VINDPM stay active. |
| **`IP_MULTICAST`, `PM_WAKELOCKS_GC`** | Both enabled in Huawei's `msm8939_user_defconfig`. Multicast (IGMP) is needed for mDNS/NSD, Cast and DLNA discovery. |
| **CFQ default I/O scheduler** | The tablet ran noop, because the device tree's `sys.io.scheduler` property does nothing on LineageOS 19.1. Huawei's kernel defaults to CFQ, the only scheduler here that honours Android's I/O priorities. |
| **JDN-W09 and JDN-AL00 DTBs** | The bootloader picks the DTB by board ID (L01 `0x1f94`, W09 `0x1f92`, AL00 `0x1fc2`); with only the L01's in `dt.img`, a W09 or AL00 finds no match. `huawei_jdn_w09.dts` and `huawei_jdn_al00.dts` include `huawei_jdn_l01.dts` and override only what Huawei's sources differ in: model, board ID, modem ID, no SAR sensor, the variant name in the identity nodes, and front porches 60/100 on the W09 panels. |
| **Touch interrupt by default** (`nvt_ts_jdn`) | The interrupt on gpio 13 works (0 interrupts idle, 235 in 10 s of swiping), and touch is clearly more responsive than polling every 15 ms. Gated on display state like polling was; polling stays as the fallback (`use_irq=0`). |
| **HI843 O-Film OTP calibration** (`camera_v2_kiwi` eeprom) | The rear module keeps its calibration (module info, white balance, lens shading, AF) in on-chip OTP, but the generic EEPROM read returned plain sensor registers, so the camera HAL ran without white-balance calibration. Huawei's reader is ported with its register tables unchanged; if the OTP read fails, userspace gets zeros rather than the probe failing. |

Checked and not ported: the GPU driver (no Huawei changes), the battery
profile and battery-ID map (identical), Huawei's charger telemetry, and
mdss CABC.

## Building

    export ARCH=arm64 SUBARCH=arm64
    export CROSS_COMPILE=<path>/aarch64-linux-android-4.9/bin/aarch64-linux-android-
    mkdir -p ../kout
    make O=../kout jdn-64_defconfig
    make O=../kout -j$(nproc) Image dtbs

GCC 4.9 is required; this kernel does not build with a modern GCC or clang.

Within a LineageOS tree it is built by the device tree's
`TARGET_KERNEL_SOURCE` (`kernel/huawei/jdn`), so `mka bootimage` is the
usual route; the device tree packs all three DTBs into `dt.img`.

## Device tree

https://github.com/Shaheddan/android_device_huawei_hwjdn

- `main` — TWRP 3.7.0 device tree (Phase 1)
- `lineage-19.1` — LineageOS 19.1 device tree
- `jdn-pad-power` — LineageOS 19.1 with the Phase 3 work, W09/AL00 support
  and the crave.io build script

## License

GPLv2, as the Linux kernel.
