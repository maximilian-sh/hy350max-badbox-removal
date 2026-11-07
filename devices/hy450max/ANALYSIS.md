# Magcubic HY450 MAX: findings and a no-flash mitigation

One unit, one firmware build (`Projector_20260821.111618`). Evidence is static analysis of the APKs (string extraction, no decompilation) plus DNS observation and a listening-port check over ADB. No dynamic analysis and no packet capture, so what the components actually send is not established. Strings in a dex file can also belong to dead code.

Companion files in this directory: `domains.txt`, `packages.txt`, `hashes.txt`.

## Device

- Sold as Magcubic HY450 MAX, Android 14 (`ro.build.version.sdk=34`).
- The device poses as a Rockchip reference board: `ro.product.model=rk3566_rgo`, `ro.product.manufacturer=rockchip`, fingerprint `rockchip/rk3566_rgo/rk3566_rgo:13/TQ2A.230305.008.F1/...` (Android 13 in the fingerprint, 14 in `ro.build.version.release`). The hardware strings are Allwinner: `ro.hardware=sun50iw15p1`, `ro.board.platform=hermes`, `ro.product.board=exdroid`. Userland is 32-bit (`armeabi-v7a`, kernel 5.15.167 `armv7l`).
- `ro.boot.verifiedbootstate=orange`, `ro.boot.flash.locked=0`, SELinux Permissive, `ro.secure=1`, `ro.adb.secure=1`. The ADB shell is the unprivileged `shell` user.
- `/system/xbin/hotacksu` is present as `-rwsr-sr-x root shell` (the setuid-root binary from the HY350 Max write-up). Not executed.
- `/system/preinstall` is empty. `/system/bin/preinstall` (4531 bytes) and `/system/bin/appsdisable` (300 bytes) exist; contents not read.

## Renamed packages (prefix `com.hy.` instead of `com.htc.`)

These are not in `iocs/packages.txt`, which lists the `com.htc.*` names. This directory adds them as separate files and changes none of the existing lists.

- `com.hy.storeos` (versionName 2609042030). Runs as uid system. Holds INSTALL_PACKAGES, DELETE_PACKAGES, INJECT_EVENTS, READ_LOGS, READ_CONTACTS, CAMERA and location (including background). Strings in classes.dex: `appstore.hanychx.com/device/app/install/report`, `.../device/channel/check`, `.../device/sign/app/list`, `.../device/sign/app/update/check`, `ota.hanychx.com/api/ota/check`, `faq.hanychx.com/static/privacy_policy/`.
- `com.hy.hyotaupdate` (versionName 2608191543). Strings: `ota.hanychx.com/api/ota/check`, `ota.hanychx.com/api/ota/upgradeResult`.
- `com.hotack.screensaversdk` (versionName 2611041809), installed under `/data/app`. Strings: `pb-api.aodintech.com/api/collections/addialog/records`, `.../adtype/records`, `.../blacklist/records` (ad-dialog configuration), plus Zeasn endpoints (`acc-saas.zeasn.tv/log/batch/report`, `acc-saas.zeasn.tv/log/open/report`). This is the process behind the `pb-api.aodintech.com` DNS query.
- `com.htc.closedialog` (versionName 2512271425). Has a BootReceiver, a RemoteControlService and `com.peasun.aispeech.aiopen.AIOpenService`. Strings: `store-api.aodintech.com/report/address`, `/upload/create`, `/upload/chunk`. What it uploads is not determined.
- `com.htc.btremote.receiver` (v3.0.0): boot receiver. Strings: `store-api.aodintech.com/sign/app/list`, `luminago.app`.
- `com.htc.magcubicos`: string `appstore.hanychx.com/device/sign/app/list`. It is the launcher and holds the projector settings; do not remove it.
- `com.htc.htcsetup_wizard_d55`: privacy-policy links on `aodintech.com`.
- The process `com.huawei.connection` is part of `com.android.toofifi` (service `com.huawei.connection.service.NearFieldManagerService`), not a separate package.

SHA-256 of the APKs on the device (also in `hashes.txt` next to this file):

```
3f0d4dfea5905b109a537d0368d8ba8da7b492c8c814f4bd78f16be3dd389876  com.hy.storeos
8887334123d450a6dc7b5d82d49688f9319b14f27d9c5686301d4f2cf4ed13ea  com.hy.hyotaupdate
6fc7c0add7f42c93d1cb09adc6bf434e20be4b77ca737900e1254dc3f353bb7b  com.hotack.screensaversdk
c71ab472f0607ddd03a62a0bfaccc80740a0e16fe1c520f07550cb06b59cc292  com.htc.closedialog
a0c6ef808e8a764b69baf35a486fda0702daecd9a32e96a5ef9b5008b7038089  com.htc.btremote.receiver
9f97199b0f7701ad57ccad361c1df41c6585d2e5e74d52e258b005642976052e  com.htc.magcubicos
bf5c63329540a80baad75547915204e6cfcb2f1aa5dd9234f747beb57f5d90d7  com.htc.htcsetup_wizard_d55
8ab67e6b161cf81d72bb068d4527e0547c3edcf0b05424ec59f15aafc6b4725a  com.android.toofifi
ae812379f220409230fb07b2278cf80fc658b2b0edab1ecf6b6394064e632c96  com.htc.samescreen
```

## Not found under the HY350 Max package names

`com.htc.storeos`, `com.htc.htcotaupdate`, `com.htc.expandsdk`, `com.htc.eventuploadservice`, `com.hotack.silentsdk`, `com.hotack.writesn`. The store, OTA and ad functionality sits in the renamed packages above.

## DNS observation

A fresh device joined an otherwise quiet network. At first boot, within about three seconds, these names were queried for the first time: `www.google.cn`, `ntp.ntsc.ac.cn`, `pb-api.aodintech.com`, `appstore.hanychx.com`, `ota.hanychx.com`, `saas.zeasn.tv`, `deviceportal.zeasn.tv`, `relay.luminago.app`, `log.saas.zeasn.tv`. The burst repeated several times later in the session, with `recommend.zeasn.tv` and `appcdn.hanychx.com` added. Between bursts: `hmma.baidu.com` roughly every six minutes for about half an hour, and `www.google.cn` every five to ten minutes. Attribution to the projector is by timing, not by source address.

## Listening ports

With the vendor casting apps enabled, `/proc/net/tcp` showed 17 listening TCP sockets, 14 of them bound to all interfaces: 1234, 5555 (adbd), 7889, 7890, 8890, 8899, 9528, 10008, 10012, 10013, 32017, 32018, 32019, 55556. After disabling `com.android.toofifi`, `com.htc.samescreen`, `com.mphotool.airplay`, `com.mphotool.dlna`, `com.softwinner.miracastReceiver` and `com.htc.btremote.receiver` together, only 1234 and 5555 remained. The owner of 1234 was not identified. Ports were not attributed to individual packages, and nothing was sent to them.

## Mitigation without flashing (unrooted, reversible)

1. Settings, About, tap Build Number seven times, Developer options, Wireless debugging, "Pair device with pairing code".
2. On the computer: `adb pair <ip>:<pairing-port> <code>`, then `adb connect <ip>:<port>` (the port shown on the main Wireless debugging screen; 5555 worked after pairing).
3. Disable for user 0 (the shell user is allowed to):

```
pm disable-user --user 0 com.hy.storeos
pm disable-user --user 0 com.hy.hyotaupdate
pm disable-user --user 0 com.hotack.screensaversdk
pm disable-user --user 0 com.htc.closedialog
```

   Optional, if you do not need casting or the bundled store: `com.android.toofifi`, `com.htc.samescreen`, `com.mphotool.airplay`, `com.mphotool.dlna`, `com.softwinner.miracastReceiver`, `com.htc.btremote.receiver`, `com.android.vending`. Disabling these removes AirPlay and Miracast.
4. Verify with `pm list packages -d`, `ps -A` and the listening-port check. Re-enable with `pm enable --user 0 <package>`.
5. Block `aodintech.com` and `hanychx.com` in DNS. Casting needs `server.mphotool.com` once.

Tested: the packages stay disabled and are not running after 20 seconds, and the listening-port result above. Not tested: persistence across a reboot or a factory reset, and whether the Bluetooth remote still works with `com.htc.btremote.receiver` disabled. The launcher and projector settings (`com.htc.magcubicos`) were left alone.

## What this does not fix

- `hotacksu` stays; anyone with an ADB shell can use it to become root.
- Vendor apps that run as uid system and stay enabled, such as `com.htc.magcubicos`.
- Hard-coded IPs bypass DNS filtering; the list in `iocs/ips.txt` needs a gateway firewall.
- A factory reset or a firmware update may re-enable the disabled packages.
- The bootloader is unlocked and SELinux is permissive on the stock image.

## Differences from the HY350 Max write-up

- Package prefix `com.hy.` for the store and OTA, and `com.hotack.screensaversdk` as an additional ad-SDK package installed under `/data/app`.
- Store and OTA run on `hanychx.com` (the write-up lists `aodintech.com` and `triplesai.com`). The `aodintech.com` endpoints still appear in other packages.
- `com.android.vending` is present at `/product/priv-app/Tubesky/Tubesky.apk` (48 MB, versionName 34.1.09-23, sha256 `4b6d959d97f0222cedce87684dd64eeefbbc0ebec141208600aaaa0d70650008`). The classes are Play Store (`finsky`) code and the package carries the Google signing certificate (SHA-256 `F0:FD:6C:5B:41:0F:25:CB:25:C3:B5:33:46:C8:97:2F:AE:30:F8:EE:74:11:DF:91:04:80:AD:6B:2D:60:DB:83`, the same as `com.google.android.gms`). It was not verified against a Google release, and Android may skip signature verification for preinstalled system apps, so "genuine" is not proven. No vendor endpoints were found in its strings.
- Play services are present on this unit (`com.google.android.gms` 22.36.16 under `/system/app/PrebuiltGmsCorePano`), unlike issue #3 (no GMS on the HY350 Max).
