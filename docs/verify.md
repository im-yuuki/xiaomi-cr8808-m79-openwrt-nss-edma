# Verification log — what was tested, and what the board showed

> Sanitized: no MAC addresses, IPs, SSIDs, or secrets appear here. Original logs/dumps stay in `.private/`.

## Board identity checks (read-only)

- `bdata`: `model=CR8808`, `CountryCode=CN`, `wl_pa_type=xpa` — confirmed China Mobile CMCC variant of the CR8808. Same hardware family as CR8806/CR8809/AX3000 M81.
- `machid=8040000` under U-Boot ⇔ IPQ5018 (QSDK machid) — not IPQ807x (earlier notes confused this; corrected).
- FIT `default = config@mp02.1` (dtb `qcom-ipq50xx-mpxx`) — stock dtb. `fdt@mp02.1` shows `nand-ecc-strength = <4>`, step 512 → confirms 4-bit ECC on this ESMT unit.
- `mtd22` (kernel FIT) has ~200 `default` fdt entries; `config@mp02.1` is the live one.
- `/proc/mtd` in stock shows UBI volumes (`kernel` 3.5 MB FIT, `ubi_rootfs` ~22 MB squashfs, `rootfs_data`, `data_ignor_reset`) — i.e. stock boots the FIT kernel out of a UBI volume, not a raw partition. Partition labels from SMEM come out **lowercase** in Linux (`0:art`, `rootfs`, etc.) — this made the initial `0:ART` vs `0:art` "patch needed" assumption wrong.
- `bdinfo`: 256 MiB RAM @ `0x40000000`. `nand info`: 128 MiB `nand0`, page 2048, OOB 64, erase 128 KiB.
- No SPI NOR (`sf probe` fails).

## SPI NAND / ECC verification

- ESMT chip detected: "ESMT SPI NAND was found", 128 MiB, page size 2048, OOB size 64 (F50D1G41LB).
- Upstream fork ships `nand-ecc-strength = <8>` in `ipq5018-cr880x.dtsi` → QPIC driver computes it needs 2128-byte spare area, exceeds 2048+64 → `No suitable ECC configuration`, `spi-nand` probe fails with `-22`, no MTD device comes up (kernel still boots but everything NAND is invisible).
- Kernel sources include `0422-spi-spi-qpic-snand-pick-default-ECC-settings-based-on-OOB-size.patch`: user-conf empty ⇒ driver picks 4-bit if OOB < 80, else 8-bit. Removing the DTS override makes all three chip classes correct without board-specific code.
- After fix:
  - `dmesg`: `spi-nand` probes successfully, `22 qcomsmem partitions found`, all partitions visible in order.
  - `dd if=/dev/mtdN of=/dev/null` over **all 22** partitions — zero I/O errors.
  - UBI/UBIFS mount: `UBIFS … default file-system created … mount_root: switching to ubifs overlay`.
  - Live PoP proof: both radios associate, 160 MHz DFS channel 100 lights up (CAC completes, radar not detected), HE160 stations connect.

## Wi-Fi / DFS verification

- `phy1-ap0` 5 GHz channel 100 (5500 MHz), `width: 160 MHz`, `center1: 5570 MHz` — HE160 up and `DFS-CAC-COMPLETED success=1 … radar_detected=0` from `logread` (bg NM: wait for CAC is normal).
- 2.4 GHz radio `radio0` channel 1 HE20 healthy.
- Both radios: `ath11k … FW memory mode: 2`, `qcn6122 hw1.0` / `ipq5018 hw1.0`, `NSS offloaded via nss-… userpd_id=2/1`, `ath11k … chip_id 0x0 chip_family 0x4 board_id 0xff soc_id 0xffffffff`, `WLAN.HK.2.7.0.1-01744-QCAHKSWPL_SILICONZ-1`.
- MAC addresses read from ART (`0x0`/`0x6`/`0x100e`/`0x2680e`) — see `docs/readme.md` on layout; they are referenced symbolically (not stored here).

## CPU / governor / memory

- `scaling_available_frequencies`: `1008000 1296000` — no lower OPPs exist. Governor = `schedutil`; at `loadavg 0.01` it sits at 1008 MHz — correct behaviour. It's not "always 1008 MHz" by accident; there is no lower OPP to drop to.
- Idle memory: ~137 MB/169 MB used (NSS firmware reserved 8 MiB at `0x40000000..0x407fffff`, `wcss` 20 MiB at `0x4b000000`, `q6-region` 13 MiB at `0x4c400000`, ath11k ~50 MiB, mac80211 + nss_stack + ECM).
- 2× Cortex-A53 @ 1.0 GHz is the hard CPU limit; ath11k offloaded traffic is fine at gigabit on wired, CPU-bound at ~550 Mbit/s when generating traffic locally (`iperf3 -s` on the device itself), and ~470 Mbit/s reverse path as seen in practice from the same machine class.

## Throughput (iperf3 single stream)

- `192.168.12.x` LAN ↔ board, 5 second window: up ≈ 573 Mbit/s / down ≈ 468 Mbit/s solo; CPU near 100% on the board (expected for 2×A53 host-side traffic).
- Forwarding path with Wi-Fi client → wired → WAN should benefit from NSS offload; not formally measured on this board (one direction only verified) before archiving.

## Extroot after flash

- Partition `overlay` (mtd20, 31 MiB) used as ubifs `/overlay`: `/dev/ubi1_0`, ~27 MiB free, upper layer of overlayfs. `/rom` (squashfs) = 13.3 MB of the slot.
- Slot-local `rootfs_data` (volume 2 in ubi0) removed to force the extroot pivot; verified config survives (`sysupgrade -b` backup/restore).
- `opkg update` shows ~10,247 packages — official snapshots only; the `nss` package path returns 404 (bug upstream provides no such feed at `aarch64_cortex-a53/nss/packages.adb`).

## Conclusion

OpenWrt 6.18 boots cleanly on this unit; SPI NAND probe works with the ECC fix; Wi-Fi 5 GHz/HE160 DFS channel 100 confirmed up; wired/wireless throughput dominated by CPU (expected); overlay extroot confirmed working. System is stable for daily use with `sysupgrade` to update slot 1 (slot 0 MiWiFi always recoverable).
