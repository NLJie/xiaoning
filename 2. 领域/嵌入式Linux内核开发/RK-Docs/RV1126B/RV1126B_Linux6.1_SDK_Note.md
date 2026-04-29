# RV1126B Linux6.1 SDK Note

---

**Contents**

[TOC]

---

## rv1126b_linux6.1_release_v1.2.0_20251220.xml Note

```
- Fix voltage adjustment issue with RV1126B/RV1126B-P PWM regulator to prevent instability problems
- Resolve crash issue when dynamically enabling/disabling multi-camera AIISP
- Add SPI NAND support for RV1126B EVB1/EVB2
- Fix screen flickering in certain scenarios with VOP when RV1126B DDR total bus width is 16-bit
- Improve ISP FPN + AIBNR performance, requiring rkaiq upgrade to v6.0x32.0 and ISP driver update to v3.2.0 or later
- Resolve unstable ADB connection issue between Linux-6.1 SDK and macOS/Ubuntu
- Update rknn-llm to v1.2.3
- Added PCBA function support in buildroot
- Added support for the 6.1.141 RT-Linux kernel patch
- Resolved the issue where u‑boot compilation no longer depends on Python 2
- Added support for the RKIPC fastboot feature for RV1126B
```

## rv1126b_linux6.1_release_v1.1.0_20250920.xml Note

```
- RV1126B OTP supports the Non-Protected OEM Zone feature, providing customers with flexible security configuration space.
- MCU documentation and basic reference code.
- AI-ISP and distortion correction function code and documentation.
- Confirmation of thermal control and DVFS frequency-voltage table updates (CPU: 1.9 GHz, NPU: 1.05 GHz).
- Support for RV1126B binocular IPC AVS fusion and stitching.
- Fix for sleep abnormalities in RV1126B 3Gb/4Gb LPDDR4(X) and potential initialization issues with some LPDDR4(X) templates.
- RKAIQ updated from v6.0x30.4 to v6.0x31.0, supporting AI-ISP switching functionality.
- Rockit updated to v2.46.0.
- RV1126B CPU performance optimization.
- Improved stability of certain LPDDR4(X) frequencies for RV1126B under high temperatures.
- Support for RV1126BJ chips.
- Support for RV1126B Buildroot Fastboot functionality.
- Support for RKAI large models on RV1126B.
- Open-source support for RV1126B Rockit OSAL.
```

## rv1126b_linux6.1_release_v1.0.0_20250620.xml Note

```
- The first release version.
```

## rv1126b_linux6.1_release_v0.2.0_20250615.xml Note

```
- Update Rockit from version v2.30.1 to v2.32.2.
- Add MCU function support for the RV1126B vacuum cleaner configuration.
- Fix the abnormal update issue.
- Fix the RKAIQ exception in Yocto.
- Update the development documents and tools related to RV1126B.
- Save and restore PVTPLL before and after system sleep and wake-up.
- Provide system reboot stability.
- Support HPMCU startup.
- Improve system thermal control stability.
```

## rv1126b_linux6.1_release_v0.1.0_20250515.xml Note

```
- Updated Rockit from v2.28 to v2.30.1:
Redefined Tile4x4 format (impacting VI/VPSS/GDC modules), resolved ISP/VPSS system crashes and VI memory leaks,
added USB camera support for VI, and disabled MPI buffer kernel virtual address mapping by default.
- Added sweeping robot configuration support for rv1126b/rv1126bp.
- Refactored rkscripts and relocated to device/rockchip/overlay directory.
- Fixed RV1126B stability issues under high/low temperature scenarios.
- Updated RV1126B-related development documentation and tools.
- Upgraded bl32 to v1.03, resolving Crypto errors in dynamic shared memory usage.
- Upgraded bl31 to v1.04, enabling LP_AOA sleep mode, enhancing memory management via three non-secure configurations
(remap_dsmc/remap_perixip/remap_pmuxip), and activating TSADC_SHUT_M0 low-power feature.
```

## rv1126b_linux6.1_release_v0.0.3_20250428.xml Note

```
- RKAIQ Updated (v6.0x30.2 → v6.0x30.2)
Enhanced SC850 AIISP algorithm model, optimized ISP35 IQ structure, and core AWB/AIBNR modules.
- Rockit Upgraded (v2.27.1 → v2.28.0)
Optimized AI-VQE auto-adaptation mechanism, GDC buffer calculation, and VPSS timeout handling.
Improved memory management (system heap adaptation, POOL retrieval fixes, VALLOC log expansion).
Fixed stability issues: initialization freezes, VO playback interruption, frame rate control failures, VDEC parameter anomalies, and RGA buffer errors.
- Added 32-bit System Support for rv1126b RKIPC
Enhanced compatibility with 32-bit environments.
- Added Aging Tests for rv1126b RKIPC
Integrated rockchip-test for stress testing and reliability validation.
- Updated CPU/NPU/ENC OPP-Tables
Enhanced stability across performance states.
- Added Thermal Zone Support for rv1126b
Implemented thermal monitoring to prevent overheating-related performance issues.
- DDR Bin Updated to v1.02
Addressed instability issues in certain DDR3/DDR4 modules.
- BL32 Updated to v1.02
Corrected ECC parameter handling during non-secure OTP reads.
```

## rv1126b_linux6.1_release_v0.0.2_20250416.xml Note

```
- Added support for modules: DDR frequency scaling, CPUIDLE, standby wake-up, rknpu, AIISP, FEC
- Updated RKAIQ to v6.0x30.2
- Fixed reboot MPP freeze issue
- Added RV1126B RKIPC support
- Implemented SecureBoot security feature support
- Added Rockit samples support
- Enabled memtester for RV1126B
- Updated Rockit to v2.27.1
- Added RKADK support
```

## rv1126b_linux6.1_release_v0.0.1_20250326.xml Note

```
- The first alpha version
```
