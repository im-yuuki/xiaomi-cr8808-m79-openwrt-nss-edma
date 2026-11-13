# cr880x-openwrt

Running **OpenWrt snapshot (kernel 6.18.60)** on the **Xiaomi CR880x** family — **CR8806 / CR8808 / CR8809 / Redmi AX3000 M81** — Qualcomm IPQ5018 + QCN6122, 256 MB RAM, 128 MB NAND (ESMT SPI NAND on the tested CR8808 unit).

This repository archives the work done to get OpenWrt (with **Qualcomm NSS hardware offload**) running on that board:

- **`docs/`** — device profile, U-Boot/console notes, build & flash guide, verification results.
- **`artifacts/SHA256SUMS.txt`** — hashes of the flashed images (reproduce, don't download).
- **`openwrt-nss-edma/`** — git submodule → **`im-yuuki/openwrt-nss-edma`**, branch `cr8808-bringup-full`. It contains a single upstream-visible fix (4-bit ECC for ESMT SPI NAND on this board) plus an optional reproducible-build overlay (`files/`) used for this device: extroot on `mtd20` (partition `overlay`, ~27 MB UBIFS `/overlay`), pin official snapshot feeds, 6 lines of `fstab`/preinit glue.

The base platform is [`kuncy7/openwrt-nss-edma`](https://github.com/kuncy7/openwrt-nss-edma) branch `c3po-tag-8021q` — the community Qualcomm NSS EDMA effort for IPQ5018 (upstream Linux 6.18, `stmmac` driver, `kmod-qca-dwmac-nss`). Neither this repo nor the fork redistribute Xiaomi proprietary blobs: everything is in-tree open-source or built by the vendor build system.

## Status

Working snapshot — boots from NAND, both radios (2.4 GHz IPQ5018 + 5 GHz QCN6122, 802.11ax/HE160 on DFS channel 100 verified), NSS wired-plane offload (`fw_mask=0x3`, ECM accelerating), ath11k Wi-Fi offload, LuCI present. Not a supported OpenWrt release; a community build for this hardware, kept as an archive of the work.

## Layout

```
.github/
docs/
  device.md        — hardware profile (summary, sanitized)
  uboot-notes.md   — U-Boot console notes + safety rules
  build-flash.md   — build from source, install, extroot, verify, rollback
  verify.md        — what was verified on hardware
artifacts/
  SHA256SUMS.txt   — image hashes (not shipping binaries)
openwrt-nss-edma/  — submodule (build tree, branch cr8808-bringup-full)
.private/          — NOT in git: dumps, nvram, logs, local notes
```

## Quick start (pointers only; details in docs/build-flash.md)

1. `git submodule update --init
`
2. Build in a Linux VM ≥ 16 GB RAM (e.g. OrbStack/Docker on macOS). See `docs/build-flash.md`.
3. Flash following `docs/build-flash.md` via U-Boot console; keep MiWiFi in **slot 0** as fallback — do not erase it unless you accept losing it.
4. After flash: expect `mount` showing `overlay /dev/ubi1_0` and `df -h` ≈ `/rom` 13 MB + `/overlay` ~27 MB free (see `docs/verify.md`).

## Working notes

- The project does **not** ship binaries of board dumps; see `.gitignore` (`.private/`).
- Commit convention: Conventional Commits.
- Device dumps/secrets never belong in git.
