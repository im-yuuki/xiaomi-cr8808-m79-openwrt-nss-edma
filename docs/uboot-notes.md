# U-Boot notes — Xiaomi CR880x (read-only findings + behavior quirks)

> **Read before doing anything.** Board-specific, not general U-Boot guidance.
> The board self-resets on data abort instead of returning errors. Erase/write commands are destructive and need explicit intent.
> Everything here was learned on a board running stock MiWiFi; these notes are not a guarantee for other units.

## What is present

- `U-Boot 2016.01 (May 15 2023)`, built with `arm-openwrt-linux-muslgnueabi-gcc` (OpenWrt-legacy Xiaomi build)
- U-Boot is **32-bit ARM** but boots **aarch64** kernels via TZ monitor (`Jumping to AARCH64 kernel via monitor`)
- Console: `serial@78AF000` (BLSP1 UART1), 115200 8N1; 120000 bps is measurably cleaner on the WCH CH34x link used in this lab
- Board is **silent** until you send a byte — passive reads return 0 bytes; send CR to reprint banner/prompt
- Prompt: `IPQ5018#`
- `machid=8040000` (board)
- `bootcmd=bootmiwifi`, `boot_wait=on`, `bootdelay=3`, `bootargs=ubi.mtd=rootfs_1 root=mtd:ubi_rootfs rootfstype=squashfs rootwait`, `machid=8040000`, `flash_type=11` (NAND)
- Slot flags: `flag_boot_rootfs` (0=slot 0, 1=slot 1), `flag_last_success`, `flag_ota_reboot`, `flag_boot_recovery`, `flag_boot_success`, `flag_boot_type`, `flag_try_sys{1,2}_failed`
- Environment lives in `0:APPSBLENV` (mtd10, 512 KiB); `reset` clears RAM env (unconfirmed by design — treat every env write as lost on reset)
- **`saveenv` writes to flash** — persisting `flag_boot_rootfs` survives reboot; limit env writes to the two boot flags and restore expectations accordingly
- `mtdparts` env is partial (`nand0:0x2400000@0x2e80000(fs)`); the full table comes from SMEM

## Behavior quirks (learned live)

- `md.l` with an unmapped address (e.g. raw NAND offsets like `0x5280000`, or even `0x44000000` outside the relocated window) → `data abort` → CPU reset → banner reprints. **`md.l` is RAM-only**; NAND must first go through `nand read <ram-addr>` then `md.l` the buffer. Only addresses inside the window near `0x4A000000` are safe to `md.l`.
- `nand erase <offset> <size>` / `nand write <ram-addr> <offset> <size>` work fine — the NAND driver in U-Boot is independent of Linux's.
- `SF (SPI flash)` probing fails — no SPI NOR on this board (only NAND).
- `nand info`: 128 MiB nand0, page 2048, OOB 64, erasesize 0x20000 (128 KiB), ECC ~QSDK-style 4/8-bit (unit-dependent).
- `smeminfo` summary shows LEB size `126976` (`0x1F000`) — matches the UBI volume erasesize shown in Linux (`mtd22`..`mtd25` are UBI volumes).

## Safe commands (all run on the live board, non-destructive)

`version`, `bdinfo`, `coninfo`, `printenv`, `help`, `sleep`, `crc32`, `cmp`, `base`, `test`, `echo`, `mii`, `i2c`, `usb`, `uart`, `nand` (without `erase`/`write`/`verify`), `ubi` (read-only usage), `mmc`, `dm`, `fdt`, `pci`, `ar8xxx_dump`, `ipq_mdio`, `ipq5018_mdio`, `btni`, `btnc`, `canary`, `bootmenu`

## Commands to never run blindly

`reset` — cold reboot (generally safe; loses RAM env; used in a controlled way during bring-up)
`saveenv` — persists env to flash; only use for the slot flags: `flag_boot_rootfs`, `flag_last_success`
`nand erase`, `nand write` — destroys/creates NAND content; use offsets from MIBIB table only
`ubi part/create/delete` — UBI volume operations; read-only usage only unless authoring a new image
`bootm` — boots image from RAM; abort if env had pending slot choice change
`tftpput` / `xqup` / `bootz` / `go` / `bootelf` / `fuseipq` — writes or executes flash directly
Anything with `rm`, `erase`, `flash`, `save` — destructive

## Slot flags cheat sheet

```
flag_boot_rootfs=0   → boot slot 0 (rootfs, mtd18)   [MiWiFi stock]
flag_boot_rootfs=1   → boot slot 1 (rootfs_1, mtd19) [OpenWrt]
flag_last_success=0  → last successful boot was slot 0
flag_last_success=1  → last successful boot was slot 1
flag_ota_reboot=1    → bootmiwifi boots the *other* slot (fallback used on OTA)
```

## Verified safe boot recipe (TFTP from host)

```
tftpboot 44000000 <image.fit>
bootm 44000000        # no #config@... needed — FIT has default configuration
```

## Quick recovery paths

- Boot fails → power cycle; U-Boot auto-boots stock from slot 0 (`flag_boot_rootfs=0`) or slot 1 (`1`)
- Broken kernel → flip flag in U-Boot (`setenv flag_boot_rootfs 1` or `0`), `saveenv`, `reset`
- Hard brick → U-Boot still intact (resides outside writable slots); re-flash via TFTP only
