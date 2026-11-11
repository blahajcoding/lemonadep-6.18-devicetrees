# redwood on Linux 6.18

Porting the Redmi Note 12 Pro 5G / POCO X5 Pro 5G (redwood, SM7325) from
Qualcomm's 6.12 kernel platform to Android Common Kernel `android17-6.18`
(6.18.33).

| Repository | What it holds |
|---|---|
| [redwood-6.18-kernel](https://github.com/suvojeet-sengupta/redwood-6.18-kernel) | ACK `android17-6.18` + Qualcomm's common delta + soc-repo, redwood fixes |
| [redwood-6.18-modules](https://github.com/suvojeet-sengupta/redwood-6.18-modules) | vendor trees (display, GPU, audio, securemsm, mm-drivers), redwood drivers, bring-up tools |
| redwood-6.18-devicetrees (this one) | the redwood devicetree, as carried over from 6.12 |

The 6.12 notes this port started from are in [README-6.12.md](README-6.12.md)
and [STATUS.md](STATUS.md).

## Status tracker

Last updated 2026-11-12. Tested in the bring-up USB shell, not in Android.

Legend: ✅ works · 🟡 loads/probes, not proven end to end · ⛔ broken ·
⬜ not started

### Kernel and core

| Item | State | Notes |
|---|---|---|
| Boot to init | ✅ | GKI built-ins kept, `lahaina_perf` only shapes modules |
| UFS storage | ✅ | `ufs-qcom` first; logdump reads/writes fine |
| USB gadget (ACM shell) | ✅ | `/dev/ttyACM*` on the PC, `rsh618.py` drives it |
| First stage modules | ✅ | 230 in-tree modules; `loadall` brings up 172 with 0 failures |
| arm-smmu | 🟡 | `pgsize_bitmap` set on the domain, resv regions skipped for the splash console, TLB flushed on every unmap and `iommu.strict=1`; deferred TLB path still needs a proper fix |
| qcom-pdc | ✅ | probe wrapper fixes a CFI type mismatch |
| Splash console | ✅ | `redwood_splashcon` built in, early boot text on the panel |
| Logdump | ✅ | log saved every 2 s, two logdump partitions rotated per boot |
| WALT | ✅ | removed, plain EAS |
| Watchdog | ⬜ | `qcom_wdt_core`/`qcom_soc_wdt` left out during bring-up |
| CoreSight | ⛔ | `coresight-funnel` resets the SoC; kept out of `loadall` |
| LPM governor | ⛔ | `qcom_lpm` panics; kept out of `loadall` |

### Peripherals

| Item | State | Notes |
|---|---|---|
| Display (msm_drm) | 🟡 | binds, finds the M20 36 02 0a DSC video panel, `card0-DSI-1`; no frame pushed from userspace yet |
| GPU (kgsl) | 🟡 | binds with GMU and SMMU; firmware (`a660_*`) only comes with /vendor |
| Audio | 🟡 | all 30 lahaina modules + `msm_ext_display` load; no sound card until ADSP runs |
| Voice DAIs | 🟡 | `msm-pcm-voice`, `msm-voip-dsp` deferred on ADSP |
| Remoteprocs (ADSP, CDSP, MPSS, WPSS) | 🟡 | registered, offline: firmware lives on vendor partitions |
| Touch | ✅ | Goodix Berlin-D (mainline driver) registers input, xiaomi_touch hooks in |
| Fingerprint | 🟡 | `goodix_fp` loads, uinput device registered |
| Speakers (aw88261) | 🟡 | both amps answer (chip id 2113) |
| Haptics | 🟡 | `aw8697-haptic` loads |
| IR blaster | ✅ | `ir-spi` registers `rc0` |
| Video (msm-vidc) | 🟡 | loads, context banks attach |
| Crypto (qce, qcedev, qcrypto, qrng) | ✅ | load, CE BAM up |
| qseecom / smcinvoke | 🟡 | load; TZ apps untested |
| ICE (inline crypto) | 🟡 | 64 keyslots, HW wrapped keys, HWKM probes; code identical to 6.12 apart from `key->raw` → `key->bytes`; never programmed with a real key |
| `tz_log` | ⛔ | probing it resets the phone (same on 6.12); not bundled |
| Wi-Fi (qcacld, cnss) | ⬜ | not built for 6.18 |
| Mobile data (IPA, rmnet) | ⬜ | dataipa, datarmnet, datarmnet-ext not built |
| Camera | ⬜ | camera kernel needs the 5.4-era techpack source |
| NFC | ⬜ | not built |
| Bluetooth | 🟡 | `btpower`, `bt_fm_slim` load; needs WPSS/firmware |

### Android

| Item | State | Notes |
|---|---|---|
| AOSPA first stage on 6.18 | ⬜ | not tried |
| /data (FBE + metadata encryption) | ⬜ | on hold: /data is shared with the daily slot |

## Test rules

- Slot b runs the daily AOSPA on 6.12. Slot a is the test slot.
- Prefer `fastboot boot` (RAM boot) with slot b left active, so nothing
  persists.
- No Android boot with /data for now. /data, metadata and misc are shared
  between slots: a 6.12 test with /data writable once left the 5.4 system's
  /data unusable (see [README-6.12.md](README-6.12.md)).
- The first stage fstab mounts `/metadata` with `formattable`; a failed
  mount there would reformat the key directory of the daily /data. Any
  Android test needs `testguard` holding userdata, misc, metadata, persist
  and qmcs read-only and `/metadata` mounted `ro` without `formattable`.

## Crashes found so far

| When | Cause | Fix |
|---|---|---|
| Reset during `loadall` | `coresight-funnel` | left out of `modules.all` |
| Panic | `qcom_lpm` governor | left out of `modules.all` |
| Reset on module load | `tz_log_dlkm` probe | not bundled |
| Early oops | arm-smmu resv regions, missing `pgsize_bitmap` | fixed in kernel |
| Stale TLB permission fault | arm-smmu deferred invalidate | TLB flush on unmap (workaround) |
| CFI panic | `qcom-pdc` probe type | probe wrapper |

## Next

1. RAM boot test image (6.18 boot args in the boot image, so slot b's
   vendor_boot works with `fastboot boot`).
2. `testguard`: also guard metadata, persist, qmcs.
3. Bring-up shell: one logdump partition for all processes; push
   `icetest`.
4. Build wlan, dataipa/rmnet, camera, NFC for 6.18.
5. Fix CoreSight and the LPM panic properly; proper arm-smmu TLB fix.
6. Only then: AOSPA first stage on 6.18 with everything shared held
   read-only.

## Building

Kernel: `redwood-6.18-modules/bringup/build-redwood618.sh` (on the build
server, under `~/r618`).

Vendor modules: `redwood-6.18-modules/bringup/build-vendor618.sh`
(mm-drivers, display, GPU, securemsm, audio). Test images:
`bringup/pack-bringup618.sh <boot.img> <vendor_boot.img>`.

Written and maintained by Suvojeet Sengupta.
