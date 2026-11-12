# cr880x-openwrt

Port OpenWrt cho **Xiaomi CR8808** (và dòng CR880x khác) — Qualcomm IPQ5018 + QCN6122, 128 MiB NAND, 256 MB RAM.

Mục tiêu: build OpenWrt mới nhất cho board này và cài đặt lên nó, thay cho firmware gốc MiWiFi.

## Trạng thái

Đang khảo sát / port. Chưa có bản build.

## Tài liệu

- [`docs/device.md`](docs/device.md) — thông tin phần cứng board (SoC, flash, phân vùng, ART, stock firmware)
- [`docs/uboot-notes.md`](docs/uboot-notes.md) — U-Boot và bảng phân vùng (read-only findings)

## Cấu trúc

```
docs/    # tài liệu công khai
.private/ # tài liệu & dump cục bộ (KHÔNG commit, xem .gitignore)
```

## License

MIT (trừ khi có ghi chú khác).
