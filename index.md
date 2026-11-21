# Xiaomi CR880x OpenWrt (NSS offload)

One directory per build. Firmware images and that build's signed APK
repository (all `kmod-*.apk`) are kept together: a kmod only installs on
an image from the same build.

| build | kmod feed | firmware |
|---|---|---|
| ipq50xx-2026.11.21-3 | [packages.adb](ipq50xx-2026.11.21-3/packages.adb) | [firmware/](ipq50xx-2026.11.21-3/firmware/) |
| ipq50xx-2026.11.20-1 | [packages.adb](ipq50xx-2026.11.20-1/packages.adb) | [firmware/](ipq50xx-2026.11.20-1/firmware/) |

On the router: `apk update && apk add kmod-wireguard`.
