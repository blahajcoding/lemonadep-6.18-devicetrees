# lemonadep devicetrees for Linux 6.18

Devicetrees for the **OnePlus 9 Pro** (`lemonadep`, SM8350/Lahaina) forward-port
onto Android Common Kernel 6.18.

This repository starts from Suvojeet Sengupta's redwood 6.18 devicetree tree
because it already carries the Qualcomm Lahaina/Yupik base devicetrees forward
to the 6.18 clock and platform bindings. lemonadep board overlays are being
ported incrementally on top of the converted Lahaina base.

## Current status

**Initial scaffold, not complete board support.**

- `qcom/lahaina-v2.1.dts` is the production base reference.
- `qcom/lemonadep-overlay.dtso` describes normal lemonadep hardware identity,
  project IDs, aliases, boot arguments and the stock ramoops layout.
- `qcom/lemonadep-t0-overlay.dtso` does the same for T0 hardware.

The following stock nodes are intentionally not present yet and must not be
assumed to work: display/panel timings, Synaptics touch, camera, TFA98xx audio
amps, OPLUS charging, NFC, fingerprint, sensors and high-current/wireless
charging. They are documented in the kernel repository's `PORTING.md` and will
be added as validated driver ports land. This is deliberate deferral, not a
silent workaround.

## Building

The devicetrees are preprocessed with Clang and compiled with the kernel's
`dtc`. Normally use the top-level build in `lemonadep-6.18-kernel`:

```sh
cd kernel/lemonadep-6.18
./build.sh dtbo
```

The module repository's `bringup/build-dt.sh` currently validates the Lahaina
base DTB and both lemonadep overlays. `mkdtboimg.py` packs both overlays into
one `dtbo.img`.

## Credits

- Qualcomm/CodeLinaro for the Lahaina devicetree base
- Suvojeet Sengupta for the 6.18 Lahaina/Yupik conversion architecture
- OnePlus/OPLUS for the stock lemonadep devicetree hardware descriptions
