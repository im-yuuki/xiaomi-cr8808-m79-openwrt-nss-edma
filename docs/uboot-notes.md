# CR8808 — U-Boot console (read-only notes)

Ghi chú từ phiên khai thác U-Boot bằng cách chỉ dùng lệnh **read-only**. Mục đích: hiểu boot flow, partition table và map bộ nhớ để port OpenWrt.

> Các lệnh `setenv`, `saveenv`, `erase`, `flash`, `bootm`, `xqup`, `tftpput`… **không** được chạy nếu chưa có ý định flash thật.

## 1. Thông tin cơ bản

| Mục | Giá trị |
|---|---|
| SoC | **IPQ5018** (machid `0x8040000`) |
| UART | `serial@78AF000` = BLSP1 UART1, 8N1 |
| Baud | 115200 (`bdinfo`) — 120000 thực tế ổn định hơn |
| RAM | 256 MiB @ `0x40000000` (bank 0, start `0x40000000`, size `0x10000000`) |
| ARMv7 | U-Boot `2016.01` (May 15 2023), build `arm-openwrt-linux-muslgnueabi-gcc` |
| Prompt | `IPQ5018#` |

**Lưu ý:** prompt `IPQ5018#` là in dấu — board **đúng là** IPQ5018 (Mach ID `0x8040000` là QSDK machid của IPQ5018; UART base `0x78af000` khớp upstream `arch/arm64/boot/dts/qcom/ipq5018.dtsi`).

## 2. NAND

```
Device 0: nand0, sector size 128 KiB
  Page size       2048 b
  OOB size          64 b
  Erase size    131072 b
  subpagesize        0 b
  options         0x0
  bbt options     0x0
```
(Đọc trực tiếp từ `nand info` trong U-Boot.)

### UBI (`smeminfo`)

```
ubi0: attached mtd1 (name "mtd=0", size 36 MiB)
ubi0: PEB size: 131072 bytes (128 KiB), LEB size: 126976 bytes
ubi0: VMD header offset: 2048 (aligned 2048), data offset: 4096
ubi0: good PEBs: 288, bad PEBs: 0, corrupted PEBs: 0
```

LEB = 126976 = `0x1F000` — đúng bằng `erasesize` 0x1f000 của các UBI volumes trong `/proc/mtd`.

## 3. Chỉ thị boot

| Env | Giá trị |
|---|---|
| `bootcmd` | `bootmiwifi` |
| `bootdelay` | `3` |
| `boot_wait` | `on` |
| `bootargs` | `ubi.mtd=rootfs_1 root=mtd:ubi_rootfs rootfstype=squashfs rootwait` |
| `mtdparts` | `nand0:0x2400000@0x2e80000(fs)` |
| `mtdids` | `nand0=nand0` |
| `flash_type` | `11` (NAND) |
| `fdt_high` / `fdtcontroladdr` | `0x4A400000` / `0x4a9d4004` |

Boot flow: `bootmiwifi` chọn slot (A/B) theo các cờ `flag_boot_*` → `ubi part fs` → `ubi read 0x%x kernel` → `bootm 0x%x#<config>` → kernel trong UBI volume `kernel`.

Cờ (đã quan sát): `flag_boot_rootfs`, `flag_boot_success`, `flag_boot_type`, `flag_try_sys1_failed`, `flag_try_sys2_failed`, `flag_last_success`, `flag_ota_reboot`, `flag_boot_recovery`, `flag_show_upgrade_info`.

## 4. Lệnh read-only (đã verify)

`version`, `bdinfo`, `coninfo`, `mtdparts`, `part list`, `flinfo`, `nand info`, `smeminfo`, `sf probe`, `printenv`, `help`, `echo`, `sleep`, `crc32`, `cmp`, `md`, `mm`, `nm`, `base`, `test`, `mii`, `i2c`, `usb`, `uart`, `nand`, `ubi`, `mmc`, `dm`, `fdt`, `pci`, `ar8xxx_dump`, `ipq_mdio`, `ipq5018_mdio`, `btni`, `btnc`, `canary`, `bootmenu`.

**Không chạy** nếu chưa rõ ý định: `reset`, `saveenv`, `setenv`, `editenv`, `run`, `bootm`, `bootelf`, `bootz`, `go`, `erase`, `flasherase`, `flash`, `fatwrite`, `protect`, `xqup`, `rm(crash)`, `fuseipq`, `tftpput`, `dhcp`, `bootmiwifi`, `bootz`.

## 5. Link chất lượng không ổn định

- Board **im lặng**: `read` thuần → 0 byte; phải gửi một ký tự (ví dụ `0x0d`) để kích board in lại banner/prompt.
- Sau reboot, đường link có thể bị chèn byte rác (ví dụ 25–35% dữ liệu hỏng ở 115200).
- Hướng dẫn: mở cổng nhiều lần, `--max-bytes` nhỏ (~900), nghỉ vài giây giữa các lần; luôn `close` cổng sau khi dùng.

## 6. Partition table đầy đủ

Xem [`docs/device.md`](./device.md).
