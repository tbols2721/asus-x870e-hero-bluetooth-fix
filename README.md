# ASUS ROG Crosshair X870E Hero Bluetooth initialization fix

A scoped firmware and driver workaround for the motherboard's onboard MediaTek
MT7927/MT6639 Bluetooth adapter, USB `0489:e13a`. On the affected Debian kernel
configurations, the device can bind to `btusb` yet remain down with an all-zero
Bluetooth address. This repository provides verified firmware extraction,
a four-line device-ID patch, guarded installation, and removal scripts.

The Bluetooth function connects internally through USB. It is the motherboard's
native adapter. These scripts concern Bluetooth; they do not modify the Wi-Fi
driver. Basic Bluetooth initialization does not require LibrePods or an AirPods
application.

## Supported configuration

Use this workaround only after matching the board, USB identity, symptoms, and
kernel scope. Check whether your current distribution already supplies the
firmware and device support before installing an override.

| Component | Supported or tested configuration |
| --- | --- |
| Motherboard | ASUS ROG Crosshair X870E Hero |
| Bluetooth hardware | MediaTek MT7927/MT6639, USB `0489:e13a` |
| Distribution | Debian 13, amd64 |
| Kernel with booted hardware testing | `6.12.115+deb13.1-amd64` |
| Additional build and DKMS installation | `6.12.116+deb13-amd64`; not boot-tested |
| Reference audio stack | BlueZ 5.82 and PulseAudio 17.0 |
| Patched module srcversion, both builds | `29AC947EAFB09F046C1A14A` |

The installer checks the board and running ABI. It refuses an enabled Secure
Boot configuration and does not configure module signing. Other kernels and
boards require a separate review; successful compilation alone is not evidence
of working controller initialization.

## Why the workaround is needed

The affected configuration has two gaps. The required Bluetooth payload is
missing, and the `btusb` quirks table lacks the board's specific device-ID
entry. The exact payload was absent from the Debian stable and backports
firmware packages inspected on 9 November 2026. That package observation is
dated evidence, not a statement about every later package version.

The controller uses Bluetooth USB interface class `0xe0`. An existing Foxconn
vendor alias for `0x0489` applies to vendor-specific class `0xff`; it does not
supply this interface's MediaTek setup. Generic class matching can bind the
interface without selecting the required initialization path.

The patch adds this entry to `quirks_table`:

```c
	/* ASUS ROG Crosshair X870E Hero, MT7927/MT6639. */
	{ USB_DEVICE(0x0489, 0xe13a), .driver_info = BTUSB_MEDIATEK |
						     BTUSB_WIDEBAND_SPEECH },
```

The tested `btmtk` module already contains MT6639 support, the chip-ID
workaround, firmware-section filtering, and the firmware path. The
[device-ID patch](patches/0001-btusb-asus-mt6639-device-id.patch) selects
existing driver logic; `btmtk` is not replaced.

## Installation

Review the scripts first. Use a local terminal and keep a non-Bluetooth input
device available: driver installation and removal reload Bluetooth and disconnect
its devices. Preparation requires network access. The driver installer obtains
DKMS and both exact kernel-header packages through APT. If the configured
repositories no longer provide those headers, reassess the kernel scope instead
of substituting an unreviewed version.

Install the preparation and diagnostic dependencies, then clone the repository:

```bash
sudo apt-get install --no-install-recommends --no-remove git python3 python3-dbus patch initramfs-tools bluez usbutils
git clone https://github.com/tbols2721/asus-x870e-hero-bluetooth-fix.git
cd asus-x870e-hero-bluetooth-fix
./scripts/check-system
```

`check-system` is read-only. It exits with status 1 while the firmware or powered
onboard controller is unavailable; this is expected before a needed fix. Inspect
its report to confirm the target hardware. It omits Bluetooth addresses and
host identity; see [PRIVACY.md](PRIVACY.md) for diagnostic-sharing guidance.

Prepare and install the firmware, then prepare and install the driver:

```bash
./scripts/prepare-mt6639-firmware
sudo ./scripts/install-mt6639-firmware
./scripts/prepare-mt6639-driver
sudo ./scripts/install-mt6639-driver
```

### Firmware preparation

The preparation script downloads the official ASUS MTK WiFi archive V5.6.0.3998
and extracts the Bluetooth payload from `mtkwlan.dat`. It treats the archive as
data and does not execute a Windows program. It verifies the archive and payload
before staging the file. The archive hash matches ASUS's published checksum.

| Artifact | SHA-256 |
| --- | --- |
| ASUS archive | `b377fffa28208bb1671a0eb219c84c62fba4cd6f92161b74e4b0909476307cc8` |
| Bluetooth payload, 688341 bytes | `669c5c99a0c59c85c1285d3d1b8b31915c2d31341a2244f4eddcbfd60ffbbc76` |

The firmware installer rechecks the payload and writes it as root-owned mode
0644 to:

```text
/usr/lib/firmware/mediatek/mt7927/BT_RAM_CODE_MT6639_2_1_hdr.bin
```

It backs up a differing existing file under
`/var/lib/airpods-xfce/firmware-backups` and updates installed initramfs images.
Firmware installation alone does not reload the driver or establish that the
controller works. Vendor binaries are not redistributed; downloads and staged
files remain under ignored `local/`.

### Driver preparation and activation

Driver preparation downloads hash-verified source files from Linux stable
v6.12.115, applies the patch, and creates a seven-file build manifest. Repeating
preparation recreates that manifest from the required source files. The
installer checks staged and root-owned copied sources, builds both target ABIs
before installing either, and installs through DKMS. BTF debug metadata is
omitted.

The DKMS package and source directory are named `airpods-btusb`, version `1.0`,
for compatibility with the companion project. The device-ID correction is not
specific to AirPods.

The installer reloads Bluetooth, checks that the resolved module comes from
DKMS, and compares its loaded srcversion with the tested value. It then finds
the adapter by USB identity, requires a nonzero address, and powers it on.
Build, installation, or load failures trigger an attempted rollback. If the
verified module loads but the controller does not initialize, the fix remains
installed and the script reports that a full power-off may be required. That
message does not establish the cause of every initialization failure.

## Verification and pairing

Check the actual loaded driver after installation:

```bash
./scripts/check-system
/usr/sbin/modinfo -n btusb
/usr/sbin/modinfo -F srcversion btusb
cat /sys/module/btusb/srcversion
/usr/sbin/dkms status -m airpods-btusb
lsusb -d 0489:e13a
```

The module path should resolve under `/lib/modules/<kernel>/updates/dkms/` or
its `/usr/lib/modules/` equivalent. Both srcversion checks should match the
table above. Reading `/sys/module` checks the loaded module; inspecting a file
on disk alone does not establish that it is active.

Controller numbering can change after reload or reboot. Match the radio to USB
`0489:e13a` instead of assuming it is always `hci0`. Select that adapter in your
Bluetooth manager, put the peripheral in discoverable pairing mode, then pair
and connect. Use an appropriate BlueZ agent for the peripheral. Before powering
off or unplugging another adapter, check whether it serves connected devices.

Confirm audio by listening in both ears. A powered controller, successful
pairing, or running audio sink alone does not establish audible output. Test
microphone input separately: the tested audio stack uses different profiles for
A2DP stereo and hands-free microphone operation.

## Validation and limits

Reference testing on 9 November 2026 verified the installed firmware hash and
ownership, DKMS module path, loaded srcversion, powered controller, and zero
receive/transmit errors at inspection. AirPods Pro 3 paired and completed
service discovery. Standard-SBC A2DP produced clear audio in both ears. A
40-second PulseAudio debug capture contained no matching bitpool, skipping,
underrun, error, or failure lines.

A separate five-second speech test through the real Bluetooth source produced
changing, unclipped 16 kHz mono input. A2DP stereo was restored afterward. These
bounded tests establish a working baseline; they do not establish full-day
reliability, suspend/resume behavior, conferencing-application input selection,
or booted hardware compatibility for the second ABI.

## Removal and future kernels

To remove the driver override:

```bash
sudo ./scripts/remove-mt6639-driver
```

DKMS archives the distribution's original module for restoration; it can move
that file out of the stock module path while the override is installed. Removal
restores the stock module through DKMS, reloads Bluetooth, and retains firmware.
On the affected stock kernels, this also removes the device-ID correction
required for onboard initialization.

Recovery handlers attempt to reload a driver and restart Bluetooth even if
cleanup fails. Failed DKMS cleanup preserves sources for diagnosis. Inspect the
reported module path and loaded state before treating recovery as complete.
Firmware removal or restoration is a separate administrative operation; there
is no automated firmware uninstaller.

`AUTOINSTALL="no"` and the exact ABI filter prevent older source from silently
overriding future kernel fixes. Before booting a later kernel, check whether
its stock driver recognizes this device and supports MT6639. If another patch
is needed, review the corresponding source and test that kernel explicitly.
Do not broaden the filter only because an untested build compiles.

## Sources, credits, and licenses

Firmware source: the [official ASUS support page](https://www.asus.com/uk/supportonly/rog%20crosshair%20x870e%20hero/helpdesk_download/)
and [MTK V5.6.0.3998 archive](https://dlcdnets.asus.com/pub/ASUS/mb/08WIRELESS/DRV_WiFi_MTK_MT7925_MT7927_TP_W11_64_V5603998_20250709R.zip).
The extraction layout was checked against
[jetm/mediatek-mt7927-dkms](https://github.com/jetm/mediatek-mt7927-dkms/blob/main/extract_firmware.py).
The preparation scripts download data directly and do not run third-party installers.

The technical explanation builds on
[Jean-François Marlière's MT6639 analysis](https://lists.openwall.net/linux-kernel/2026/02/08/473),
[Javier Tia / jetm's MT7927 work](https://github.com/jetm/mediatek-mt7927-dkms),
[shaunstanislauslau's board-specific investigation](https://github.com/shaunstanislauslau/x870e-hero-linux-bluetooth-fix),
and the Linux Bluetooth maintainers' code in
[btusb.c](https://github.com/gregkh/linux/blob/v6.12.115/drivers/bluetooth/btusb.c)
and [btmtk.c](https://github.com/gregkh/linux/blob/v6.12.115/drivers/bluetooth/btmtk.c).

Original automation and documentation use the [MIT license](LICENSE). The
Linux device-ID patch uses [GPL-2.0-only](LICENSES/GPL-2.0-only.txt). Downloaded
firmware remains subject to vendor terms and is not included in this repository.

The separate [XFCE AirPods integration](https://github.com/tbols2721/xfce-airpods)
builds on [Kavish Devar's LibrePods](https://github.com/librepods-org/librepods)
and [GM's omarchy-pods](https://github.com/thisisgm/omarchy-pods). Those projects
provide AirPods controls and are optional for basic controller initialization.
