# Xiaomi CR8808 — Device profile (public notes)

Tài liệu này mô tả phần cứng và firmware gốc của **Xiaomi CR8808** (bản Trung Quốc, model `CR8808` — biến thể China Mobile). Dùng làm mốc khi port OpenWrt cho thiết bị này.

> Các dump NAND và giá trị cụ thể của cá nhân (MAC, IP, SSID, keys) chỉ nằm trong `.private/stock-backup/` (không public). File này giữ lại phần kỹ thuật có thể công khai.

## 1. Phần cứng

| Mục | Giá trị |
|---|---|
| SoC | Qualcomm **IPQ5018** (machid `0x8040000`) |
| Wi-Fi | 2.4 GHz: IPQ5018 (ath11k); 5 GHz: **QCN6122** (ath11k) |
| RAM | **256 MiB** @ `0x40000000` |
| NAND | **128 MiB** `nand0`, page 2048 B, OOB 64 B, erase 128 KiB |
| SPI NOR | Không có |
| Switch | QCA8337 (WAN + 3 LAN, conduit = GMAC1) |
| FEM 2.4 GHz | FEM loại "xPA" (FEM của bản CR8808; khác `ipa`/`epa` của Redmi AX3000 — cần BDF khớp) |

## 2. Boot chain

```
SBL1 → MIBIB → QSEE/DEVCFG/CDT → APPSBL(_1) = U-Boot 2016.01
                                   → bootmiwifi → FIT kernel (UBI volume `kernel`) → ubi_rootfs
```

- U-Boot **32-bit ARM** (biên dịch bằng `arm-openwrt-linux-muslgnueabi-gcc` 5.5.0), **có hỗ trợ boot kernel 64-bit** qua TZ monitor (`Jumping to AARCH64 kernel via monitor`). FIT `config@mp02.1` = AP-MP02.1 (`qcom,ipq5018-mp02.1`).
- `boot_wait=on` đã được bật; `bootdelay=3`; `bootcmd=bootmiwifi`.
- `bootargs=ubi.mtd=rootfs_1 root=mtd:ubi_rootfs rootfstype=squashfs rootwait`, `mtdparts=nand0:0x2400000@0x2e80000(fs)`, `flash_type=11` (NAND), `machid=8040000`.
- Cờ A/B: `flag_boot_rootfs` (0 = slot 0 / rootfs, 1 = slot 1 / rootfs_1), `flag_boot_success`, `flag_boot_type`, `flag_try_sys{1,2}_failed`, `flag_last_success`, `flag_ota_reboot`, `flag_boot_recovery`, `flag_show_upgrade_info`.
- Đã bật `uart_en=1`, `ssh_en=1`, `telnet_en=1`, `boot_wait=on` (user tự bật).
- `bootmiwifi` đọc UBI volume tên `kernel` từ partition `fs` và boot FIT tương ứng; `bootm 0x%x#%s` với `config@mp02.1`.

## 3. Bảng phân vùng NAND (block = 128 KiB)

`0:MIBIB` chứa hai bảng phân vùng giống hệt nhau, lặp lại ở 2 vị trí (redundancy). Bảng chính (đơn vị = block `0x20000`):

| Partition | start block | blocks | Offset | Size |
|---|---|---|---|---|
| `0:SBL1` | 0x0 | 4 | 0x000000 | 512 KiB |
| `0:MIBIB` | 0x4 | 4 | 0x080000 | 512 KiB |
| `0:BOOTCONFIG` | 0x8 | 2 | 0x100000 | 256 KiB |
| `0:BOOTCONFIG1` | 0xa | 2 | 0x140000 | 256 KiB |
| `0:QSEE` | 0xc | 8 | 0x180000 | 1 MiB |
| `0:QSEE_1` | 0x14 | 8 | 0x280000 | 1 MiB |
| `0:DEVCFG` | 0x1c | 2 | 0x380000 | 256 KiB |
| `0:DEVCFG_1` | 0x1e | 2 | 0x3c0000 | 256 KiB |
| `0:CDT` | 0x20 | 2 | 0x400000 | 256 KiB |
| `0:CDT_1` | 0x22 | 2 | 0x440000 | 256 KiB |
| `0:APPSBLENV` | 0x24 | 4 | 0x480000 | 512 KiB |
| `0:APPSBL` | 0x28 | 0xa | 0x500000 | 1280 KiB |
| `0:APPSBL_1` | 0x32 | 0xa | 0x640000 | 1280 KiB |
| **`0:ART`** | 0x3c | 8 | 0x780000 | **1 MiB** |
| `0:TRAINING` | 0x44 | 4 | 0x880000 | 512 KiB |
| `bdata` | 0x48 | 4 | 0x900000 | 512 KiB |
| `crash` | 0x4c | 4 | 0x980000 | 512 KiB |
| `crash_log` | 0x50 | 4 | 0xa00000 | 512 KiB |
| **`rootfs`** | 0x54 | 0x120 | 0xa80000 | 36 MiB |
| **`rootfs_1`** | 0x174 | 0x120 | 0x2e80000 | 36 MiB |
| `overlay` | 0x294 | 0xf8 | 0x5280000 | 31 MiB |
| `data` | 0x38c | 0x6c | 0x7180000 | 13.5 MiB |

`/proc/mtd` trên thiết bị cho thấy thêm:
- `kernel` (3.5 MB, FIT, `Linux-4.4.60`, dtb `qcom-ipq50xx-mpxx`),
- `ubi_rootfs` (~22 MB, Squashfs xz),
- `rootfs_data` (~27 MB, UBI),
- `data_ignor_reset` (~10.5 MB, UBI).

Các partition này có `erase size = 0x1f000` (= `0x20000 − 0x1000`) → chúng là **UBI volumes** xuất hiện như MTD.

```
UBI volume layout (rootfs slot đang chạy):
  kernel           ← FIT image (kernel + dtb AP-MP02.1)
  ubi_rootfs       ← Squashfs (OpenWrt 18.06 snapshot, LEDE ipq/ipq50xx, arm_cortex-a7)
  rootfs_data      ← overlay (rỗng khi stock)
  data_ignor_reset ← (rỗng khi stock)
```

## 4. ART partition (`0:ART`, mtd13, 1 MiB)

Phần offset quan trọng (giá trị cụ thể phụ thuộc máy, đây là layout):

| Offset | Nội dung |
|---|---|
| `0x000` | MAC eth0 |
| `0x006` | MAC eth1 |
| `0x1000`–`0x1fff` | **2.4 GHz caldata** (`0x20000` bytes) |
| `0x26800`–`0x467ff` | **5 GHz caldata** (`0x20000` bytes) |

→ `caldata_extract "0:ART" 0x1000 0x20000` (2.4G), `caldata_extract "0:ART" 0x26800 0x20000` (5G).
**Label trong MIBIB là chữ HOA: `0:ART`** — lưu ý khi viết DTS/caldata script (nhiều DTS dùng `0:art`).

## 5. Stock firmware

- File: `miwifi_cr8808_all_fac_release_6.4.52.bin`
- Nền tảng: **OpenWrt 18.06 snapshot (LEDE)**, board `ipq/ipq50xx`, `arm_cortex-a7`, kernel **4.4.60** (build "ARM OpenWrt FIT" của Xiaomi — QSDK 11.x style).
- Rootfs: banner "XiaoQiang", có LuCI, `cnssdaemon`, `vasdocker`, `mosquitto`, thư viện `andlink` (Alibaba Cloud IoT), `easymesh` — đây là MiWiFi CR8808 cho CMCC.
- Hiện tại board chạy **stock MiWiFi 6.4.52**, boot từ slot tương ứng theo `flag_boot_rootfs`.

## 6. Serial / console

- WCH CH34x USB-UART (VID `1a86`, PID `5523`).
- Baud mặc định theo `coninfo`/`bdinfo`: **115200** — nhưng đường link thường nhiễu; **120000** cho kết quả sạch hơn.
- Board **im lặng** ở prompt; phải gõ một phím thì nó in lại banner/prompt.

## 7. Ghi chú cho ai muốn port OpenWrt

- Board giống hệt **Redmi AX3000 / Xiaomi CR8806 / CR8809** về phần cứng (IPQ5018 + QCN6122 + QCA8337 + 256 MB RAM + 128 MB NAND) — chỉ khác vendor firmware/style và FEM 2.4 GHz (`wl_pa_type=xpa`).
- U-Boot 32-bit ARM nhưng **có hỗ trợ nhảy vào kernel 64-bit** qua TZ monitor — không cần thay U-Boot nếu dùng đúng FIT image.
- `qcom,smem-part` (upstream `CONFIG_MTD_QCOMSMEM_PARTS=y`) sẽ đọc partition table từ SMEM → DTS không cần hardcode offsets.
- Board data (BDF) board `xiaomi_cr8806` (board-id `255`, variant `Xiaomi-CR8806`) đã được xác minh hoạt động trên CR8808 (cộng đồng WRT).
- ART label là `0:ART` (hoa) — caldata extraction của một số DTS/scripts dùng `0:art` sẽ fail.
