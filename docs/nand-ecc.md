# NAND ECC fix for CR880x (the one board-specific patch)

## Symptom

With the default `ipq5018-cr880x.dtsi` from `kuncy7/openwrt-nss-edma` (`c3po-tag-8021q`), the SPI NAND probe on this board fails:

```
qcom_snand 79b0000.spi: Current ECC settings require 2128 bytes,
but the flash only has 2048+64 bytes.
spi-nand spi0.0: probe with driver spi-nand failed with error -22
```

No MTD devices are registered; the board cannot read/write/upgrade anything.

## Root cause

`target/linux/qualcommax/dts/ipq5018-cr880x.dtsi` hardcodes:

```
nand-ecc-step-size = <512>;
nand-ecc-strength = <8>;
```

The driver (`spi-qpic-snand`) then requires 2128 bytes of spare area (2048 + 80), which does not fit the 64-byte OOB of the **ESMT F50D1G41LB** found on this unit (pn: 1 Gb). The same tree carries `patches-6.18/0422-spi-spi-qpic-snand-pick-default-ECC-settings-based-on-OOB-size.patch`, which selects 4-bit for OOB < 80 and 8-bit otherwise — meaning simply **removing the hard override** yields the right result for both OOB classes (this unit and the OOB ≥ 80 units upstream tested).

## Fix used in this archive (see patches/)

`0001-qualcommax-ipq50xx-use-4-bit-NAND-ECC-on-CR880x.patch` — forces 4-bit/512 in the dtsi. Chosen because it was (a) the minimal change to unblock this unit, and (b) verified end-to-end here. For upstream/merge the preferred change would be **removing the hardcode entirely and letting 0422 pick ECC from OOB size** — cleaner for the whole CR8806/CR8808/CR8809 family.

## Verification (this unit)

- `dmesg`: `spi-nand spi0.0: ESMT SPI NAND was found`, `128 MiB, page 2048, OOB 64`, no ECC error, `22 qcomsmem partitions found`, all partitions visible.
- `dd if=/dev/mtdN of=/dev/null bs=1M` over all 22 partitions — zero I/O errors.
- UBI attaches, UBIFS mounts, 5 GHz HE160 channel 100 associates (`DFS-CAC-COMPLETED success=1`), both radios probe.
- Stock dtb (`fdt@mp02.1` from the boot FIT) shows `nand-ecc-strength = <4>` — consistent with the fix.

## How to apply to upstream

In the root of an `openwrt-nss-edma` checkout:

```bash
git apply /path/to/0001-qualcommax-ipq50xx-use-4-bit-NAND-ECC-on-CR880x.patch
```

Or cherry-pick the commit from `patches/*` manually (the patch text is the exact diff; title = `qualcommax: ipq50xx: use 4-bit NAND ECC on CR880x`).

## Known caveat

If your CR8806/CR8808/CR8809 board variant uses an OOB ≥ 80 chip (Winbond/Macronix 80–128 base), the **hardcoded 8** in upstream is correct for it, and the 4-bit patch in this archive's `patches/` would break it. Prefer pushing the "remove the hard override" variant, not this one, for a general fix.
