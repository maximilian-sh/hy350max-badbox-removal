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
(an AI assistant) driving the analysis, the firmware unpacking/repacking, and the
debugging. I'm publishing the notes as-is.

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
| RAM / Storage | 2 GB / 32 GB |
| Correct firmware base | **h726_p1** (board `h726-p1`, "BT control"), build `Projector_20260113.175241` |

> ⚠️ **Model matters.** `h726_p1` = HY350 Max. `h726_p2` = HY300 Max / HY300 Pro.
> Flashing the wrong board's image is how you brick. The IR remote works fine after
> flashing — p1 firmware contains IR support (`cir_param` in DTB, `sunxi-ir` in super,
> plus `ble_remote`/`bt_remote`).

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

> **Removing `com.htc.storeos` breaks the download chain.** You only need to delete ~6
> preinstalled packages to neutralize the whole thing.

See [`iocs/`](iocs/) for the full machine-readable indicator lists.

### Preinstalled malicious packages (remove these)

```
com.htc.storeos                    — silent app installer  (THE dropper — kills the chain)
com.htc.htcotaupdate               — unencrypted OTA, reinstalls malware
com.htc.expandsdk                  — ad injection + persistence
com.htc.eventuploadservice         — telemetry exfiltration
com.hotack.writesn                 — proxy helper
com.hotack.silentsdk               — SOAX proxy agent (loaded at runtime via StoreOS)
com.htc.htclauncherhighenglishd08  — malicious launcher (if present)
```

Occurrence counts in `super.fex` (via `grep -a`, your build may differ slightly):
`storeos` 12–14×, `htcotaupdate` 8–11×, `aodintech` 5×, `expandsdk` 3×,
`eventuploadservice` 2×, `hotack.writesn` 1×, `hotack.silentsdk` 0× (runtime only).

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

### 1. Remove the malware (debloat)

```bash
# Unpack the Allwinner image, then extract the super partition
#   awimg.py extract <update.img> super.fex
simg2img super.fex super.raw        # sparse -> raw, if needed
lpunpack super.raw ./super_out      # -> system_a, vendor_a, product_a (ext4)

# Delete the malware APKs WITHOUT mounting (works unprivileged in Docker).
# The preinstalled packages live under product (product_a). Use debugfs:
debugfs -w -R "rm /app/StoreOs_QZ/StoreOs_QZ.apk"       product_a.img
debugfs -w -R "rm /app/HtcOtaUpdate_QZ/HtcOtaUpdate_QZ.apk" product_a.img
debugfs -w -R "rm /app/Tubesky/Tubesky.apk"             product_a.img
# ...repeat for each malicious package directory you find (ls via debugfs -R "ls -l /app")

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
#   awimg.py replace <update.img> super  super.new.fex
#   awimg.py replace <update.img> vbmeta vbmeta.new.fex
# (checksums are recomputed automatically on repack)
```

### 4. Bypass the version lock

The bootloader refuses to flash an image with an equal/older build date. Bump it:

```
sunxi_version.fex  date field  ->  set to a future date (e.g. 2027-12-31) via dd
```

### 5. Flash via USB

1. Put the repacked `update.img` in an `/update/` folder on a FAT32 USB stick.
2. Insert into the projector's USB-A port.
3. Hold the **Power** button until the LED goes **red → blue** — flashing starts.

---

## Verifying the clean

- **Pi-hole / DNS:** after cleaning, the projector should make **0 requests** to the C2
  domains in [`iocs/domains.txt`](iocs/domains.txt). The author confirmed 0 C2 requests
  post-flash.
- Remote (BT/IR) still works.
- Apps can be sideloaded normally.

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

**Gotcha — partition full:** `/system` was 100% full ("No space left on device"). The
fix was an **in-place byte edit** of existing lines (`ro.secure=1` → `ro.secure=0`,
`ro.adb.secure=1` → `ro.adb.secure=0`) to keep the file size identical, appending only
what fit. Each reflash also needs the `sunxi_version.fex` date bumped again.

---

## Network containment (recommended regardless)

Even after cleaning, isolate the projector:

- Put it on a dedicated **IoT VLAN**; **block IoT → main LAN**.
- Point its DHCP DNS at a **Pi-hole** and load the [`iocs/pihole-denylist.txt`](iocs/pihole-denylist.txt).
- Set Pi-hole `FTLCONF_dns_listeningMode=all` so cross-VLAN queries are answered.
- Optional: firewall-block `IoT → Internet :53/:853` to stop DNS bypass, and block the
  hardcoded C2 IPs in [`iocs/ips.txt`](iocs/ips.txt) at the gateway (DNS won't catch those).

---

## Credits / sources

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
