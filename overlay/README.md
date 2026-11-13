# Reproducible-build overlay

These are the two local patches applied on top of `kuncy7/openwrt-nss-edma` (`c3po-tag-8021q`) to reproduce the images in this archive:

```
patches/0001-qualcommax-ipq50xx-use-4-bit-NAND-ECC-on-CR880x.patch   # DTS: 4-bit ECC for ESMT F50D1G41LB
patches/0001-files-overlay-for-reproducible-CR8808-build.patch        # files/ overlay for OpenWrt image
```

`overlay/files/` contains the same files, pre-copied:

- `files/lib/preinit/09_mount_overlay_data` — attach partition `overlay` (mtd20) as ubi1 at preinit; `rootfs_data` volume valid; drop the small one in `rootfs_1` so NAND extroot wins.
- `files/etc/config/fstab` — `option target '/overlay'`, `option device '/dev/ubi1_0'`, `option fstype 'ubifs'`.
- `files/etc/apk/repositories.d/distfeeds.list` — official snapshot feeds, drop the nonexistent `nss` path.
- `files/etc/config/dhcp`, `...` — no board-specific changes here.

## Apply

```bash
cd openwrt-nss-edma
git apply ../patches/0001-qualcommax-ipq50xx-use-4-bit-NAND-ECC-on-CR880x.patch
git apply ../patches/0001-files-overlay-for-reproducible-CR8808-build.patch
```

Then build with the config in `docs/build-flash.md`.
