# ASUS ROG Crosshair X870E Hero Bluetooth initialization fix

This repository documents and scripts a tested correction for the onboard
MediaTek MT7927/MT6639 Bluetooth adapter on an ASUS ROG Crosshair X870E Hero.
On the tested Debian 13 installation, the controller appeared as USB device
`0489:e13a` but remained down with an all-zero Bluetooth address. Installing
its missing firmware and adding one device-ID entry to `btusb` allowed it to
initialize, pair, and carry clear stereo audio.

The Bluetooth function connects internally through USB. It is the motherboard's
native adapter, not an external USB dongle. This correction concerns Bluetooth;
it does not install or modify the Wi-Fi driver. The successful test retained
Debian 13, XFCE, BlueZ, and PulseAudio. Changing desktop environments or moving
to Debian unstable was unnecessary in this verified case.

## Scope and evidence

This is a narrow workaround for one board and two exact Debian kernel ABIs.
The installer checks the board and running kernel before making changes.
Secure Boot was disabled on the tested system. The installer refuses an
enabled Secure Boot configuration; it does not configure module signing.

| Component | Tested configuration or result |
| --- | --- |
| Motherboard | ASUS ROG Crosshair X870E Hero |
| Bluetooth hardware | MediaTek MT7927/MT6639, USB `0489:e13a` |
| Distribution | Debian 13, amd64 |
| Running kernel tested with hardware | `6.12.115+deb13.1-amd64` |
| Additional build and DKMS installation | `6.12.116+deb13-amd64` |
| Audio stack | BlueZ 5.82 and PulseAudio 17.0 |
| Patched module srcversion, both builds | `29AC947EAFB09F046C1A14A` |

On 9 November 2026, live verification confirmed the firmware's installed hash
and root ownership, the DKMS module's resolved path, and the loaded module's
srcversion. The onboard controller became powered and `UP RUNNING`, with a
nonzero address and zero receive/transmit errors at inspection. AirPods Pro 3
paired successfully, became trusted, and completed service discovery.

A standard-SBC A2DP stream then produced clear audio in both ears, confirmed
by the listener. A 40-second PulseAudio debug capture contained no matching
bitpool, skipping, underrun, error, or failure lines. This is evidence of
successful initialization and a bounded playback test, not a long-duration
reliability or suspend/resume certification. The second kernel was built and
installed but had not received the same booted hardware test.

An external Realtek adapter had previously delivered intermittent playback.
Its underlying problem was not established. This repository does not claim
that it was defective or that radio interference caused its symptoms.

## Why the controller did not initialize

Two separate requirements were missing. The required Bluetooth firmware was
not installed and was absent from the Debian stable and backports firmware
packages inspected for this case. The installed `btusb` driver also lacked
the board's specific device-ID entry in its quirks table. That table tells
the driver which device-specific initialization path to use.

The controller's USB interface uses Bluetooth class `0xe0`. An existing
Foxconn vendor alias for `0x0489` applies to vendor-specific class `0xff`, so
that alias does not supply the missing MediaTek initialization. Generic
Bluetooth class matching can bind the interface without selecting MediaTek
setup. The correction adds this entry to `quirks_table`:

```c
	/* ASUS ROG Crosshair X870E Hero, MT7927/MT6639. */
	{ USB_DEVICE(0x0489, 0xe13a), .driver_info = BTUSB_MEDIATEK |
						     BTUSB_WIDEBAND_SPEECH },

```

The installed `btmtk` module already contained MT6639 support, the chip-ID
workaround, the required firmware path, and firmware-section filtering. No
replacement of `btmtk` was necessary. The
[four-line patch](patches/0001-btusb-asus-mt6639-device-id.patch) selects existing
logic in `btusb`; it changes no other device entries or driver logic.

## Installation

Review the scripts before running them. Use a local terminal and keep an input
device available that does not depend on Bluetooth: driver reload disconnects
Bluetooth devices. The preparation scripts need network access, Python 3,
and `patch`. The driver installer obtains DKMS and both exact kernel-header
packages through APT. If those old package versions are no longer available
from your configured Debian repositories, stop and reassess the kernel scope
rather than substituting an unreviewed kernel version.

From the checked-out repository, run the read-only system check first:

```bash
cd asus-x870e-hero-bluetooth-fix
sudo apt-get install --no-install-recommends --no-remove python3 python3-dbus patch initramfs-tools bluez usbutils
./scripts/check-system
```

Prepare and install the firmware before preparing and installing the driver:

```bash
./scripts/prepare-mt6639-firmware
sudo ./scripts/install-mt6639-firmware
./scripts/prepare-mt6639-driver
sudo ./scripts/install-mt6639-driver
```

The firmware preparation script downloads the official ASUS MTK WiFi driver
archive V5.6.0.3998 and extracts the Bluetooth payload from `mtkwlan.dat` as
data. It does not execute a Windows program. It verifies both the archive
and extracted payload before staging the file. The archive hash was checked
against ASUS's published checksum.

| Artifact | SHA-256 |
| --- | --- |
| ASUS archive | `b377fffa28208bb1671a0eb219c84c62fba4cd6f92161b74e4b0909476307cc8` |
| Bluetooth payload, 688341 bytes | `669c5c99a0c59c85c1285d3d1b8b31915c2d31341a2244f4eddcbfd60ffbbc76` |

The firmware installer checks the payload again and writes it as root-owned
mode `0644` to:

```text
/usr/lib/firmware/mediatek/mt7927/BT_RAM_CODE_MT6639_2_1_hdr.bin
```

It backs up any differing existing file and updates installed initramfs
images. Firmware installation alone does not reload the driver or establish
that the controller works. Vendor firmware is not redistributed in this
repository; downloaded and extracted files remain under ignored `local/`.

Driver preparation downloads SHA-256-verified source files from upstream
Linux v6.12.115, applies the patch, and creates a manifest of the required
build files. The installer verifies the staged and copied sources, builds
both target ABIs before installing either, and installs through DKMS. BTF
debug metadata is omitted. The DKMS package is named `airpods-btusb`, version
`1.0`, for compatibility with the original verified installation; the device-ID
correction itself is not specific to AirPods.

The installer reloads Bluetooth, checks that the resolved module comes from
DKMS, and compares the loaded srcversion with the tested value. It then finds
the onboard controller by USB identity, requires a nonzero Bluetooth address,
and powers it on. Build, installation, or load failures trigger an attempted
rollback. If the verified module loads but the controller still cannot
initialize, the fix is retained and the installer reports that a full
power-off may be required before retrying. That conditional recovery is not
a guarantee that every initialization failure is a power-state issue.

## Verify the actual adapter and audio path

Run the read-only check again after installation:

```bash
./scripts/check-system
uname -r
/usr/sbin/modinfo -n btusb
/usr/sbin/modinfo -F srcversion btusb
cat /sys/module/btusb/srcversion
/usr/sbin/dkms status -m airpods-btusb
lsusb -d 0489:e13a
```

For a supported kernel, `modinfo -n btusb` should resolve under
`/lib/modules/<kernel>/updates/dkms/` or the equivalent `/usr/lib/modules/`
path. Both srcversion checks should report
`29AC947EAFB09F046C1A14A`. Reading `/sys/module` checks the module actually
loaded in the kernel; inspecting the file on disk alone is insufficient.

Do not assume that the onboard radio is always `hci0`. It had that index in
the successful test, but enumeration can change after reload or reboot. Match
the controller to USB `0489:e13a` before selecting it in a Bluetooth manager.
Then pair the headphones explicitly through that controller and select its
audio output. If isolating another adapter, first check whether it serves
connected devices before powering it off or unplugging it.

Pairing requires the headphones to be discoverable. An initial
`AuthenticationRejected` result in the test was resolved by putting the
AirPods into their white-flashing pairing state and retrying with a registered
default agent. This pairing step is separate from kernel initialization.
For other peripherals, follow their pairing instructions and use an
appropriate BlueZ agent.

Confirm audio by listening in both ears. A powered controller, a successful
pairing, or a `RUNNING` audio sink alone does not prove audible output. Test
the microphone separately if needed: A2DP stereo playback and hands-free
microphone operation use different profiles. A subsequent five-second speech
test through the onboard adapter produced a changing, unclipped signal from
the real Bluetooth input at 16 kHz mono. Stereo was restored afterward.
Conferencing-application and suspend/resume behavior remain untested.

## Rollback and future kernels

To remove the driver override:

```bash
sudo ./scripts/remove-mt6639-driver
```

The override is installed in the DKMS module location. DKMS archives the
distribution's original module for later restoration; on the tested system
it moved that file out of the original module path. The scripts do not manually
overwrite that file. Removal deletes
the registered override and source, reloads the stock driver, and retains
the verified firmware. With the affected stock kernels, the onboard adapter
will again lack its MediaTek device-ID correction.

The recovery handlers attempt to reload a driver and restart Bluetooth even
if an operation fails. If DKMS cleanup fails, its source is retained for
diagnosis. Inspect the reported module path and loaded state before treating
a recovery attempt as successful. Firmware removal is a separate operation;
the driver removal script does not restore or delete firmware.

DKMS automatic installation is deliberately disabled with `AUTOINSTALL="no"`.
The source and installer accept only the two recorded ABIs. This prevents
older driver source from silently overriding changes in a future kernel.
Before booting a later kernel, check whether its stock driver already has
this board's device-ID entry and adequate MT6639 support. If another patch
is necessary, review the corresponding source and test that kernel explicitly.
Do not broaden the version filter merely because an untested build compiles.

## Sources and credits

The firmware comes from the
[official ASUS motherboard support page](https://www.asus.com/uk/supportonly/rog%20crosshair%20x870e%20hero/helpdesk_download/)
and its
[MTK V5.6.0.3998 archive](https://dlcdnets.asus.com/pub/ASUS/mb/08WIRELESS/DRV_WiFi_MTK_MT7925_MT7927_TP_W11_64_V5603998_20250709R.zip).
The extraction container layout was checked against
[jetm/mediatek-mt7927-dkms](https://github.com/jetm/mediatek-mt7927-dkms/blob/main/extract_firmware.py).
No third-party installation script was executed.

The investigation also drew on
[Jean-François Marlière's MT6639 analysis](https://lists.openwall.net/linux-kernel/2026/02/08/473),
[Javier Tia / jetm's upstream support work](https://github.com/jetm/mediatek-mt7927-dkms),
and the board-specific research in
[shaunstanislauslau/x870e-hero-linux-bluetooth-fix](https://github.com/shaunstanislauslau/x870e-hero-linux-bluetooth-fix).

The driver correction uses
[upstream Linux v6.12.115 btusb.c](https://github.com/gregkh/linux/blob/v6.12.115/drivers/bluetooth/btusb.c)
and the existing
[btmtk implementation](https://github.com/gregkh/linux/blob/v6.12.115/drivers/bluetooth/btmtk.c).
This repository publishes the correction and reproducible preparation scripts,
not firmware, compiled modules, personal pairing data, or raw diagnostic logs.

The separate [XFCE AirPods integration](https://github.com/tbols2721/xfce-airpods)
credits and builds on [Kavish Devar's LibrePods](https://github.com/librepods-org/librepods)
and [GM's omarchy-pods](https://github.com/thisisgm/omarchy-pods). Those projects
provide AirPods features; they are not required for this board's basic Bluetooth
initialization fix.
