# Magcubic HY350 Max — BADBOX 2.0 Malware Removal

Reverse-engineering notes and a **reproducible firmware-cleaning recipe** for the
**Magcubic HY350 Max** projector (Allwinner **H726**, Android 14), which ships with
**BADBOX 2.0** malware baked into the factory firmware.

> **No firmware image is distributed here, on purpose.** This repo documents the
> *method* in enough detail that you can clean your own stock image. Redistributing a
> modified OEM image is a copyright problem, and a pre-rooted image is a security
> downgrade for anyone who flashes it blindly. Build your own from your own dump.

---

## Disclaimer

This reverse-engineering work was done **entirely with [Claude](https://claude.com/claude-code)**
driving the analysis, the firmware unpacking/repacking, and the debugging. I'm
publishing the notes as-is.

**I take no responsibility for anything that happens to your device or network.** This
worked on my unit; it may brick yours. There are no guarantees that any of this is
correct, safe, or applicable to your firmware revision. Verify everything yourself.

I'm sharing it only because it might **point someone in the right direction** if they're
stuck on the same projector. It is a trail of breadcrumbs, not a polished tool.

> **One thing that may save you a lot of time:** without the **baked-in ADB activation
> in the custom image** (the `ro.adb.secure=0` / root step described far below), I was
> **never able to get ADB working at all** on this device. The daemon listens on 5555
> and accepts the TCP connection, but the authorization popup is suppressed and the
> handshake just hangs — no pairing code ever appears on screen. If you're fighting
> "ADB connects but won't authorize," that's why. The fix had to go into the firmware.

---

## ⚠️ Read before you do anything

- Flashing firmware **can brick your device.** You do this at your own risk.
- The author's recovery path was a **PhoenixSuit (Windows)** reflash. `sunxi-fel` on
  macOS did **not** detect the H726 — do not assume FEL recovery will save you.
- This is a **defensive / debloat** writeup. The goal is removing malware, not adding
  capabilities. The optional root step below **weakens device security** — see the
  warning in that section.

---

## Affected device

| | |
|---|---|
| Model | Magcubic HY350 Max |
| SoC | Allwinner **H726** (Quad Cortex-A53) |
| OS | Android 14 |
| RAM / Storage | 2 GB / 32 GB (author), 16 GB also confirmed — same image |
| Correct firmware base | **h726_p1** (board `h726-p1`, "BT control"), build `Projector_20260113.175241` |

### Confirmed units

| Unit | Storage | Build on the device | Vendor image cleaned | Result |
|---|---|---|---|---|
| Author | 32 GB | January 2026 build | `Projector_20260113.175241` | works |
| [#1](https://github.com/maximilian-sh/hy350max-badbox-removal/issues/1) | 16 GB | `Projector_20251224.164521` | `Projector_20260113.184230` (sha256 `e6ca3607cbddc5e6b2cbeaa6702692b723d2dd7a4632895cfef1dc5820fd4441`) | works |

The 16 GB unit takes the same image: in `sys_partition.fex`, `UDISK` has no fixed size
and takes the rest of the eMMC (the fixed partitions add up to ~2.66 GiB). Note the
vendor's January image has shown up under two build names from the same day — record the
sha256 of the image you clean.

> ⚠️ **Model matters.** `h726_p1` = HY350 Max. `h726_p2` = HY300 Max / HY300 Pro.
> Flashing the wrong board's image is how you brick. The IR remote works fine after
> flashing — p1 firmware contains IR support (`cir_param` in DTB, `sunxi-ir` in super,
> plus `ble_remote`/`bt_remote`).

### Where to get the stock firmware

You supply your own stock image (this repo distributes no binaries). The vendor
publishes them here:

- **HY350 Max (h726_p1):** <https://www.magcubic-support.com/newsinfo/3145143.html>
- **All firmware / downloads index:** <https://www.magcubic-support.com/xw>

> The vendor image is the **malware-laden stock** — that's the input you clean with the
> recipe below, not something to flash as-is. Confirm the build matches your board
> before flashing.

---

## What the malware is

The device is part of the **BADBOX 2.0** botnet. It identifies as an `ADT-3` device and:

- runs as a **residential proxy node** (SOAX network) — strangers route traffic through your home IP
- exfiltrates device telemetry
- silently installs additional apps
- has an **OTA mechanism to reinstall itself** after factory resets
- **survives factory resets** — it's in the firmware, not user space

The runtime payload (`com.hotack.silentsdk`) is **not preinstalled** — it is **pulled at
runtime by StoreOS**. This is the key insight:

> **Removing `com.htc.storeos` breaks the download chain.** Both confirmed units were
> cleaned by deleting just **three preinstalled APKs**.

See [`iocs/`](iocs/) for the full machine-readable indicator lists.

### Preinstalled malicious APKs (remove these)

These exist as real APK files in `product_a`:

```
/app/StoreOs_QZ        com.htc.storeos       — silent app installer  (THE dropper — kills the chain)
/app/HtcOtaUpdate_QZ   com.htc.htcotaupdate  — unencrypted OTA, reinstalls malware
/priv-app/Tubesky      com.android.vending   — fake Play Store (declares Google's package name)
```

Because Tubesky occupies that package name, there is no real Google Play on this device —
`com.android.vending` here is the vendor's store.

### Referenced, but not preinstalled

The other package names from the public BADBOX reports appear **only as strings** — in
`/vendor/etc/awbms_config` and inside `MagcubicOS.apk`. The `20260113.184230` image checked
in [#1](https://github.com/maximilian-sh/hy350max-badbox-removal/issues/1) has no APK for them:

```
com.htc.expandsdk                  — ad injection + persistence
com.htc.eventuploadservice         — telemetry exfiltration
com.hotack.writesn                 — proxy helper
com.hotack.silentsdk               — SOAX proxy agent (pulled at runtime via StoreOS)
com.htc.htclauncherhighenglishd08  — malicious launcher
```

If your build does contain APKs for any of these, delete them too.

> ⚠️ **Do not delete `MagcubicOS.apk` (`com.htc.magcubicos`)** even though it contains
> these strings. It is the vendor launcher and holds the projector settings: keystone
> (`CorrectionActivity`), projection mode (`ProjectActivity`), `InitAngleActivity` and the
> rest under `.activity.settings.*`. Keep it installed; if you want another launcher, set
> it as default with `cmd package set-home-activity`.

### Root backdoor: `hotacksu`

`/system/xbin/hotacksu` (in `system_a`) is an `su` binary named after the same vendor as
`com.hotack.*`. With SELinux permissive, `hotacksu 0 id` run as the **shell** user
(uid 2000) returns root; from an app uid the exec is denied by file permissions. See the
optional removal step in the recipe.

Occurrence counts in `super.fex` (via `grep -a -c`, your build may differ slightly):
`storeos` 12–14×, `htcotaupdate` 8–11×, `aodintech` 5×, `expandsdk` 3×,
`eventuploadservice` 2×, `hotack.writesn` 1×, `hotack.silentsdk` 0× (runtime only).
These identify a stock image; they are **not** a removal check (see
[Verifying the clean](#verifying-the-clean)).

---

## H726 partition layout (differs from the H713 reports!)

If you came here from an H713/HY300 Pro guide, note the differences:

- **No `oem` partition** → the popular `/oem/customer.prop` root trick does **not** apply.
- A/B seamless: `boot_a/b`, `vendor_boot_a/b`, `init_boot_a/b`, `super`,
  `vbmeta(_system/_vendor)_a/b`, `misc`, `frp`, `metadata`
- `super.fex` (~1.93 GB) = dynamic partitions (system / vendor / product) → use `lpunpack`
- **SELinux = permissive** (in vendor_boot) — helps with modification
- **AVB signed** (SHA256_RSA2048), `vbmeta` **flags = 0** (verification active)

---

## The cleaning recipe

You need: your **stock h726_p1 image**, a Linux box or Docker, `lptools`
(`lpunpack`/`lpmake`), `debugfs` (e2fsprogs), `simg2img`/`img2simg`, and an Allwinner
image (un)packer (the author used a small `awimg.py` with `list/extract/replace/repack`).

> **`awimg.py` is not published in this repo** — the commands below show its usage. Any
> Allwinner image (un)packer that recomputes the `V*.fex` checksums will do. If you write
> your own, two details cost time
> (from [#1](https://github.com/maximilian-sh/hy350max-badbox-removal/issues/1)):
>
> - Entry offsets are **aligned up to 1024 bytes**.
> - Each `V*.fex` file holds the checksum of its partner file: the **sum of 32-bit
>   little-endian words** (zero-padded at the end) **mod 2³²**. This reproduces all 13
>   stock `V*.fex` files.
>
> Test your tool with a stock → stock repack and no replacements: the output must be
> **byte-identical** to the input.

### 1. Remove the malware (debloat)

```bash
# Unpack the Allwinner image. NOTE: the second argument is an output DIRECTORY,
# not a file to select -- all ~57 .fex files are written into it.
#   awimg.py extract <update.img> <output_dir/>
./awimg.py extract update.img ./unpacked/

# super.fex is an Android sparse image (magic 3aff26ed); confirm, then convert
od -A d -t x1 -N 4 ./unpacked/super.fex
simg2img ./unpacked/super.fex super.raw   # sparse -> raw
lpunpack super.raw ./super_out            # -> system_a, vendor_a, product_a (ext4)

# Delete the malware APKs WITHOUT mounting (works unprivileged in Docker).
# The preinstalled packages live under product (product_a). Use debugfs.
# Look around first: debugfs -R "ls -l /app" product_a.img  (same for /priv-app)
debugfs -w -R "rm /app/StoreOs_QZ/StoreOs_QZ.apk"           product_a.img
debugfs -w -R "rm /app/HtcOtaUpdate_QZ/HtcOtaUpdate_QZ.apk" product_a.img
debugfs -w -R "rm /priv-app/Tubesky/Tubesky.apk"            product_a.img   # priv-app, not app
# ...repeat for any other malicious package directory you find

# Optional, untested: remove the hotacksu root binary from system_a.
# Nobody has confirmed yet that nothing on the device depends on it.
debugfs -w -R "rm /system/xbin/hotacksu" system_a.img

# Repack super
lpmake ... ./super_out -> super.new.fex      # or dd the modified partition back
img2simg super.new.raw super.new.fex         # raw -> sparse, if needed
```

> The author's first pass mounted `product_a`, deleted **StoreOs_QZ,
> HtcOtaUpdate_QZ and Tubesky**, then wrote the partition back into `super.raw`
> with `dd` (at sector offset `2682880`) and re-sparsed with `img2simg`. The
> `debugfs` route above does the same thing without needing root/mount.

### 2. Disable AVB verification so the modified super boots

`vbmeta` carries flags that must be flipped or the bootloader rejects your changes:

```
vbmeta flags: 0 -> 3      (verification + hashtree disabled)
byte offset 123 = 0x03    (offsets 120–123 hold the flags field)
```

### 3. Put the modified partitions back

```bash
# Signature: replace <image.img> <filename> <new_file> <out.img>
# <filename> is the name as printed by `awimg.py list` -- i.e. "super.fex", not "super".
./awimg.py replace update.img super.fex  super.new.fex  out.img
./awimg.py replace out.img    vbmeta.fex vbmeta.new.fex out2.img
# (checksums are recomputed automatically on repack)
```

### 4. Bypass the version lock

The bootloader refuses to flash an image whose build date is **equal to or older than
the build currently installed on the device** -- the *installed* build, not the build of
the image you downloaded. If your projector already runs a newer build than the vendor
page offers, you must clear *that* date.

`sunxi_version.fex` is a plain 40-byte ASCII file: two identical lines of exactly 20
bytes each. No `dd` offsets needed -- just rewrite it, keeping the length at 40:

```bash
# original content, for reference:
#   2026-01-13 18:48:12\n2026-01-13 18:48:12\n
printf '2027-12-31 23:59:59\n2027-12-31 23:59:59\n' > sunxi_version_new.fex
wc -c sunxi_version_new.fex        # must be exactly 40
```

Then `awimg.py replace` it in like any other file. Every reflash needs a fresh bump.

### 5. Flash via USB

1. Put the repacked `update.img` in an `/update/` folder on a FAT32 USB stick.
   Confirmed working: **MBR partition table with a single FAT32 partition**. A GPT stick
   with an EFI partition first is untested — reformat it to be safe.
2. Insert into the projector's USB-A port.
3. Hold the **Power** button until the LED goes **red → blue** — flashing starts.

Notes:

- **The flash wipes `/data`** — apps, settings and activations are gone afterwards
  (see [After flashing](#after-flashing)).
- **It may hang at 100%.** On the second unit the progress bar filled and the projector
  then sat there for about an hour without rebooting. Pulling the stick and
  power-cycling booted it normally. Only do this once the bar is completely full —
  never interrupt power while it is still writing.

---

## Verifying the clean

- **On the image, before flashing:** list the package directories — no `.apk` may be
  left in them (the directories themselves stay, that's fine). Do **not** use
  `grep -a -c` counts for this: `debugfs rm` unlinks the files but leaves their data
  blocks, so the strings are still in `super` and the counts do not drop.

  ```bash
  for d in /app/StoreOs_QZ /app/HtcOtaUpdate_QZ /priv-app/Tubesky; do
    echo "== $d"; debugfs -R "ls -l $d" product_a.img
  done
  ```

- **Pi-hole / DNS:** after cleaning, the projector should make **no requests** to the C2
  domains in [`iocs/domains.txt`](iocs/domains.txt). The author saw 0 C2 requests
  post-flash. The second unit saw a **single `store-api.aodintech.com` query** in the
  first 20 minutes, and none for `ota-api`, `cdn-store`, `pb-api`, `triplesai` or
  `leiniao`. Which process made it is not known yet — keep the DNS block in place.
- Remote (BT/IR) still works.
- Apps can be sideloaded normally.

---

## After flashing

- **Casting (AirPlay / DLNA) stays dead until it re-activates.** The receivers
  (`com.mphotool.airplay`, `com.mphotool.dlna`) stay closed and logcat repeats
  `ProcActive: getActivated() isActived=0`. The activation lived in `/data`, which the
  flash wipes, and it is re-fetched from `server.mphotool.com`. If `mphotool.com` is on
  your blocklist, allow `server.mphotool.com` for ~30 seconds, then block it again. The
  activation is stored in `settings global ft_activated_data` and casting keeps working.
  The cloud session is opened by `com.htc.samescreen` (uid 1000), not by the receivers.
- **Projector settings** (keystone, projection mode, etc.) live in `com.htc.magcubicos`
  — keep it, see above. With root ADB you can open the screens directly with
  `am start -n com.htc.magcubicos/<activity>`; `dumpsys package com.htc.magcubicos`
  lists the activity names.
- **Home Assistant:** the build has no Android TV Remote Service, so only the ADB-based
  `androidtv` integration works.

---

## (Optional) Pre-baked root ADB — ⚠️ security downgrade, do not ship to others

After removing the malware store, the author found app installation blocked because the
OEM suppressed the ADB authorization popup. The workaround was baking root ADB into
`system_a.img:/system/build.prop`:

```
ro.secure=0                 # adbd runs as root
ro.adb.secure=0             # NO authorization prompt — anyone on the LAN can connect
ro.debuggable=1
service.adb.tcp.port=5555   # adbd listens from boot
persist.adb.tcp.port=5555
```

> 🚨 **This makes the device a wide-open, unauthenticated root-ADB box on your network.**
> Only do this on an **isolated VLAN**. Never distribute an image built this way — you'd
> be handing people a worse problem than the malware. If you don't specifically need it,
> **skip this step** and keep ADB authorization on.

**Gotcha — partition full:** on the author's image `/system` was 100% full ("No space
left on device"). The fix was an **in-place byte edit** of existing lines
(`ro.secure=1` → `ro.secure=0`, `ro.adb.secure=1` → `ro.adb.secure=0`) to keep the file
size identical, appending only what fit. The `Projector_20260113.184230` image had 928
free 4K blocks, so there the lines could simply be appended. Check with
`debugfs -R "stats" system_a.img` (`Free blocks`). Each reflash also needs the
`sunxi_version.fex` date bumped again.

**Gotcha — `debugfs write` loses mode and SELinux label.** Replacing `build.prop` with
`rm` + `write` creates a new inode: the file mode (stock is `0600`) and the
`security.selinux` xattr are **not** carried over. Save both before you replace the file
and put them back afterwards:

```bash
debugfs -R "stat /system/build.prop" system_a.img                       # note Mode / Uid / Gid
debugfs -R "ea_get -f label.bin /system/build.prop security.selinux" system_a.img

debugfs -w -R "rm /system/build.prop" system_a.img
debugfs -w -R "write build.prop /system/build.prop" system_a.img
debugfs -w -R "sif /system/build.prop mode 0100600" system_a.img        # regular file, 0600
debugfs -w -R "ea_set -f label.bin /system/build.prop security.selinux" system_a.img

debugfs -R "stat /system/build.prop" system_a.img                       # compare with before
```

Uid/Gid must still be 0 (fix with `sif ... uid 0` / `sif ... gid 0` if not).

---

## Network containment (recommended regardless)

Even after cleaning, isolate the projector:

- Put it on a dedicated **IoT VLAN**; **block IoT → main LAN**.
- Point its DHCP DNS at a **Pi-hole** and load the [`iocs/pihole-denylist.txt`](iocs/pihole-denylist.txt).
  Unverified candidates are listed separately at the end of that file. Don't block
  `mphotool.com` permanently before casting has re-activated (see
  [After flashing](#after-flashing)).
- Set Pi-hole `FTLCONF_dns_listeningMode=all` so cross-VLAN queries are answered.
- Optional: firewall-block `IoT → Internet :53/:853` to stop DNS bypass, and block the
  hardcoded C2 IPs in [`iocs/ips.txt`](iocs/ips.txt) at the gateway (DNS won't catch those).

---

## Credits / sources

- [ssw112](https://github.com/ssw112) — second confirmed unit (16 GB), Allwinner checksum
  details, `hotacksu`, casting and debugfs findings ([#1](https://github.com/maximilian-sh/hy350max-badbox-removal/issues/1))
- HUMAN Security — BADBOX 2.0 research
- `Kavan00/Android-Projector-C2-Malware` (IOC report)
- `micha102/hy300pro-debloat`
- Zane St. John — "Reverse Engineering with Claude Code"
- probonopd — HY300_PRO gist
- XDA — H726 rooting guide (Magcubic HY300 Pro / Allwinner H726 / Android 14)
- linux-sunxi.org — FEL; `linux-sunxi/sunxi-tools`

## License

Findings and documentation: CC BY 4.0. Scripts/snippets: MIT. **No OEM firmware
binaries are included or redistributed.**
