# NSS feed (ipq50xx-2026.11.11/256m) — keys and verification

The nss-packages kmod feed for ipq50xx isn't signed by any key shipped in the upstream image, but each **release** ships a throwaway-built `public-key.pem` in `/etc/apk/keys/` that signs that specific feed index. The key for `ipq50xx-2026.11.11/256m` is included here so this archive is self-contained.

## Key (public) — for `ipq50xx-2026.11.11/256m`

```
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEbG8lQFi/Z2RYPB5y1rOKl6T+gVtb
hlMY8l0l3le5whTdT8WhD6KVPrbA09bLtTecFrxnKEe5jiI3NEfen5qDkw==
-----END PUBLIC KEY-----
```

Contents of this fork release are not all identical to stock OpenWrt snapshots for the same kernel, so the repo also carries the (unchanged) `distfeeds.list` pinned to official snapshots.

## To use the feed (on the router)

```sh
# 1. install the public key (name only matters for tracking)
curl -sS -o /etc/apk/keys/openwrt-nss-edma-ipq50xx-2026.11.11.pem \
  https://github.com/im-yuuki/xiaomi-cr8808-m79-openwrt-nss-edma/raw/main/artifacts/keys/openwrt-nss-edma-ipq50xx-2026.11.11.pem

# 2. add the feed
echo 'https://kuncy7.github.io/openwrt-nss-edma/ipq50xx-2026.11.11/256m/packages.adb' \
  >> /etc/apk/repositories.d/customfeeds.list

# 3. verify
apk update
```

## What it enables

```sh
apk add kmod-wireguard wireguard-tools luci-proto-wireguard
```
(and other NSS kmods built for this kernel/vermagic combination)

## Notes / gotchas

- The key verifies **only** `ipq50xx-2026.11.11/256m/packages.adb`; `std`, older releases, and snapshots-aligned builds are all verified with a different, unpublished, build-local key.
- Each release regenerates `public-key.pem` per build job; the key is intentionally not published by the upstream author.
- The feed's index URL is `ipq50xx-…/256m/packages.adb` (256 MB boards) — `std` uses `std/packages.adb` and has a different public key.
- The feed is signed only with the throwaway-built key of that specific build; for releases not built by this archive, rebuild from source instead of switching feeds.
- This file is for reference/use; verify keys by hand before trusting them.
