# redwood on Linux 6.18

Porting the Redmi Note 12 Pro 5G / POCO X5 Pro 5G (redwood, SM7325) from
Qualcomm's 6.12 kernel platform to Android Common Kernel `android17-6.18`
(6.18.33).

| Repository | What it holds |
|---|---|
| [redwood-6.18-kernel](https://github.com/suvojeet-sengupta/redwood-6.18-kernel) | ACK `android17-6.18` + Qualcomm's common delta + soc-repo, redwood fixes |
| [redwood-6.18-modules](https://github.com/suvojeet-sengupta/redwood-6.18-modules) | vendor trees (display, GPU, audio, securemsm, mm-drivers, camera, video, wlan, data), redwood drivers, bring-up tools |
| [device_xiaomi_redwood_clo `6.18`](https://github.com/suvojeet-sengupta/device_xiaomi_redwood_clo/tree/6.18) | the Android device tree for 6.18 |
| redwood-6.18-devicetrees (this one) | the redwood devicetree, as carried over from 6.12 |

The 6.12 notes this port started from are in [README-6.12.md](README-6.12.md)
and [STATUS.md](STATUS.md).

## Status tracker

Last updated 2026-11-12. AOSPA has booted to the home screen on 6.18 as a
RAM boot with /data on tmpfs; the rest is tested in the bring-up USB shell.

Legend: ✅ works · 🟡 loads/probes, not proven end to end · ⛔ broken ·
⬜ not started

### Kernel and core

| Item | State | Notes |
|---|---|---|
| Boot to init | ✅ | GKI built-ins kept, `lahaina_perf` only shapes modules |
| UFS storage | ✅ | `ufs-qcom` first; needs `IOMMU_IO_PGTABLE_FAST` (without fastmap, apps read stale zeros: SIGILL in odex, dex2oat crashes) |
| USB gadget (ACM shell) | ✅ | `/dev/ttyACM*` on the PC, `rsh618.py` drives it; the shell reopens ttyGS0 after `loadall` resets USB |
| Modules | ✅ | `loadall` loads 181 with 0 failures |
| arm-smmu | ✅ | `pgsize_bitmap` set on the domain; device tree reserved regions work again (`of_iommu_get_resv_regions` fixed for 6.18's `of_match_bus`); TLB flushed on every unmap with `iommu.strict=1` until the deferred path is fixed |
| Rust | ✅ | rustc 1.88.0, bindgen 0.72.1; ashmem (Rust only on 6.18) and the Rust Binder module; `MODVERSIONS` off |
| SELinux netlink | 🟡 | `redwood.nlmsg_legacy` falls back to the pre-xperm netlink check when vendor policy lacks `allowxperm` rules |
| qcom-pdc | ✅ | probe wrapper fixes a CFI type mismatch |
| Splash console | ✅ | off by default, `redwood_splashcon.enable=1` for bring-up; stops and unmaps when `msm_drm` loads |
| Logdump | ✅ | `bootlog.ko` writes the kernel log through bios (SELinux can't stop it), `logcap.ko` adds logcat and boot phases |
| WALT | ✅ | removed, plain EAS |
| Watchdog | ⬜ | `qcom_wdt_core`/`qcom_soc_wdt` left out during bring-up |
| CoreSight | ⛔ | `coresight-funnel` resets the SoC; kept out of `loadall` |
| LPM governor | ⛔ | `qcom_lpm` panics; kept out of `loadall` |

### Peripherals

| Item | State | Notes |
|---|---|---|
| Display (msm_drm) | ✅ | AOSPA UI on the M20 panel, 1080x2400 |
| GPU (kgsl) | ✅ | renders the AOSPA UI |
| Touch | ✅ | Goodix Berlin-D (mainline driver), usable in AOSPA |
| Charging | ✅ | status bar shows charging |
| Modem (MPSS) | ✅ | boots, baseband version shown, registers on the network |
| ADSP, WPSS | ✅ | boot from `/vendor/firmware_mnt` |
| CDSP | ⛔ | `cdsp.mdt` not found when it is requested in first stage |
| Bluetooth | ✅ | works in AOSPA |
| Wi-Fi (qcacld, icnss2) | 🟡 | built and loads; the WPSS crashed on the first copy engine access (`ccpm_pcie.c: invalid PCIe read access`) because the WLAN IOVA range started at 0 instead of the 0xa0000000 window. Fixed in the kernel (453892b), retest pending |
| Mobile data (IPA, rmnet) | ⛔ | IPA and rmnet load, no `rmnet_data` links: netmgrd's `RTM_NEWLINK` was denied by netlink xperm checks; device tree policy and the kernel fallback are in, still no links |
| Camera | ⛔ | ICP firmware prints its version, then `cam_icp_mgr_send_fw_init` times out (-110); may be the same missing reserved regions as Wi-Fi, retest pending |
| Video (msm-vidc) | ✅ | codec sessions open: empty vb2 wait hooks, `V4L2_FL_USES_V4L2_FH` kept, controls wired to the instance |
| Audio | 🟡 | lahaina modules load, sound card registers; not checked in AOSPA |
| Sensors | ⛔ | sensors registry in persist not read yet |
| Fingerprint | 🟡 | `goodix_fp` loads |
| Speakers (aw88261) | 🟡 | both amps answer (chip id 2113) |
| Haptics | 🟡 | `aw8697-haptic` loads |
| IR blaster | ✅ | `ir-spi` registers `rc0` |
| Crypto (qce, qcedev, qcrypto, qrng) | ✅ | load, CE BAM up |
| qseecom / smcinvoke | ✅ | keymaster and gatekeeper work in AOSPA |
| `tz_log` | ⛔ | probing it resets the phone (same on 6.12); not bundled |
| NFC | ⬜ | not built |

### Android

| Item | State | Notes |
|---|---|---|
| AOSPA on 6.18 (RAM boot) | ✅ | home screen, adb; first boot on an empty /data takes about 6 minutes |
| AOSPA on 6.18 with the real /data | ⬜ | `pack-daily618.sh` builds the image; not tested yet |
| Flashable build from the device tree | ⬜ | needs a `device_xiaomi_redwood-kernel-6.18` prebuilt |

## Test rules

- Slot b runs the daily AOSPA on 6.12. Slot a is the test slot.
- Prefer `fastboot boot` (RAM boot), so nothing persists. Set slot b
  active again afterwards.
- AOSPA test boots run with /data on tmpfs, and `testguard` holds userdata
  and misc read-only; metadata and persist are mounted `ro,noload`, qmcs
  `ro`.
- **Gatekeeper lives in the TEE and both slots share it.** On an empty
  /data, gatekeeperd sees no `/data/misc/gatekeeper/.coldboot` and wipes
  every enrolled user once setup completes or a credential is set, and the
  daily slot then rejects the right PIN. That happened on 2026-11-12 and the
  daily data could not be recovered. Every Android test boot with a
  non-daily /data must create `.coldboot` or set `ro.gsid.image_running=1`,
  and nobody finishes setup or sets a lock in a test boot.

## Crashes found so far

| When | Cause | Fix |
|---|---|---|
| Reset during `loadall` | `coresight-funnel` | left out of `modules.all` |
| Panic | `qcom_lpm` governor | left out of `modules.all` |
| Reset on module load | `tz_log_dlkm` probe | not bundled |
| Oops while arm-smmu registers | `of_iommu_get_resv_regions` used a NULL translation | fixed in kernel (453892b) |
| Stale TLB permission fault | arm-smmu deferred invalidate | TLB flush on unmap (workaround) |
| CFI panic | `qcom-pdc` probe type | probe wrapper |
| Oopses at `0xff000000` everywhere | splash console kept drawing into freed splash memory | stop and unmap on `msm_drm` |
| SIGILL in odex, dex2oat crashes | UFS DMA without Qualcomm's fastmap | `IOMMU_IO_PGTABLE_FAST=y` |
| zygote restart loop | no `/dev/ashmem` (Rust only on 6.18) | `CONFIG_RUST`, ashmem, Rust Binder |
| Camera refcount crash | `cam_smmu` double put of a dma-buf | `get_dma_buf` |
| WPSS fatal error after mode request | WLAN IOVAs outside the firmware's window | reserved regions back (453892b) |

## Next

1. Retest Wi-Fi and the camera ICP with the reserved regions back.
2. Mobile data: get the `RTM_NEWLINK` error from netmgrd and rmnet.
3. CDSP firmware and the sensors registry.
4. Test the daily image on the real /data, then the
   `device_xiaomi_redwood-kernel-6.18` prebuilt and a setup branch for
   flashable builds.
5. Fix CoreSight and the LPM panic properly; proper arm-smmu TLB fix.

## Building

On the build server, under `~/r618`, after `. env.sh` (it puts the clang
and Rust toolchains in `PATH`; without Rust in `PATH`, Kconfig silently
turns `CONFIG_RUST` off and ashmem and the Rust Binder with it).

Kernel: `$KMAKE Image modules`. Redwood's own drivers:
`redwood-6.18-modules/bringup/build-redwood618.sh`; vendor modules:
`build-vendor618.sh` (mm-drivers, display, GPU, securemsm, audio),
`build-wlan618.sh`, `build-data618.sh`, `build-misc618.sh`.

Test images: `repack618.sh` builds the bring-up shell image and the AOSPA
RAM boot image; `pack-daily618.sh <boot.img> <vendor_boot.img>` builds a
6.18 boot image for a slot's own system and /data.

Written and maintained by Suvojeet Sengupta.
