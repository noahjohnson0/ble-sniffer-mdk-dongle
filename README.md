# BLE Packet Capture with a Makerdiary nRF52840-MDK-USB-Dongle (GeekPi rebrand)

This is a walkthrough of getting a "GeekPi BLE adapter" (which turned out to be a
[Makerdiary nRF52840-MDK-USB-Dongle](https://wiki.makerdiary.com/nrf52840-mdk-usb-dongle/))
working as a BLE packet sniffer on macOS, producing Wireshark-compatible PCAPs.

The dongle ships preflashed with Nordic's **connectivity firmware**
(VID `0x1915` / PID `0xc00a`) for use with `pc-ble-driver`. We replace that with
Nordic's **nRF Sniffer for Bluetooth LE 4.1.1** firmware and capture link-layer
BLE traffic.

> Platform: macOS (Apple Silicon), Homebrew, Python 3.11 venv.

---

## TL;DR

```bash
# 1. Tools
brew install --cask nrfutil           # modern nrfutil
xattr -d com.apple.quarantine /opt/homebrew/bin/nrfutil   # unsigned binary
nrfutil install ble-sniffer nrf5sdk-tools

# 2. Get sniffer firmware
curl -fsSL -o /tmp/sniffer.zip \
  https://nsscprodmedia.blob.core.windows.net/prod/software-and-other-downloads/desktop-software/nrf-sniffer/sw/nrf_sniffer_for_bluetooth_le_4.1.1.zip
unzip /tmp/sniffer.zip -d ~/nrf-sniffer

# 3. Put dongle into UF2 bootloader (unplug, hold reset button, plug back in)
#    Expect /Volumes/UF2BOOT to appear and 2 green LEDs

# 4. Convert hex → UF2 (Adafruit nRF52840 family ID)
curl -fsSL -o /tmp/uf2conv.py \
  https://raw.githubusercontent.com/microsoft/uf2/master/utils/uf2conv.py
python3 /tmp/uf2conv.py --family 0xADA52840 \
  --convert ~/nrf-sniffer/hex/sniffer_nrf52840dongle_nrf52840_4.1.1.hex \
  --output ~/nrf-sniffer/sniffer.uf2

# 5. Drag-flash
cp ~/nrf-sniffer/sniffer.uf2 /Volumes/UF2BOOT/
# Dongle reboots; nrfutil device list now shows
#   Product: nRF Sniffer for Bluetooth LE

# 6. Capture
nrfutil ble-sniffer sniff \
  --port /dev/cu.usbmodem101 \
  --output-pcap-file ble_capture.pcap
```

---

## What I learned the hard way

### The dongle is *not* a Nordic PCA10059
GeekPi sells these as generic "USB BLE adapters." Mine reported as Nordic VID/PID
in `system_profiler SPUSBDataType` but actually carries the
**Makerdiary nRF52840-MDK-USB-Dongle**. You can only tell once you enter the
bootloader (hold reset, plug in) — the UF2 mass-storage volume's `INFO_UF2.TXT`
identifies the board:

```
UF2 Bootloader 0.7.1 ...
Model: Makerdiary nRF52840 MDK USB Dongle
Board-ID: nRF52840-MDK-USB-DONGLE
```

### `nordicDfu` trait ≠ buttonless DFU works
`nrfutil device list` reported `Traits: nordicDfu, nordicUsb, …`, which fooled me
into thinking I could trigger DFU over USB. **The trait is just inferred from the
USB VID** — the running connectivity firmware doesn't actually implement Nordic's
buttonless DFU service. Result:

```
Error: Internal sdfu error: Wait device connected failed:
  Device event timeout, waiting for event SerialArrived
```

The fix is physical: unplug, hold the small reset button, plug back in, release.
You'll see two solid green LEDs and a `UF2BOOT` USB drive mount.

### UF2 family IDs matter
`uf2conv.py` defaults to the Microsoft family table. The Makerdiary MDK
bootloader expects **Adafruit's nRF52840 family ID `0xADA52840`**, not Microsoft's
`0x1b57745f`. Wrong family ID → file just sits in `UF2BOOT` and never flashes.
Right family ID → the bootloader rejects the still-open file handle mid-copy
(you'll see `cp: fcopyfile failed: Input/output error` — this is success), then
reboots into the new firmware.

### Legacy `nrfutil pkg` is broken on Python 3
The PyPI `nrfutil==5.2.0` package has unfixed Py2-isms
(`.iteritems()`, `xrange`). Don't bother patching — for DFU package generation
use:
- `nrfutil nrf5sdk-tools pkg generate ...` from modern nrfutil 8.x, or
- `adafruit-nrfutil` (Adafruit's maintained fork).

Both are moot if you have a UF2 bootloader — you don't need a DFU zip at all.

### Modern nrfutil binary is unsigned on macOS
The Homebrew cask installs without code signing. macOS Gatekeeper blocks it on
first run ("Apple could not verify..."). Click **Done** (not Move to Trash), then:

```bash
xattr -d com.apple.quarantine /opt/homebrew/bin/nrfutil
```

---

## Verifying the capture

```bash
$ file ble_capture.pcap
ble_capture.pcap: pcap capture file, microseconds ts (big-endian) - version 2.4
  (Nordic Semiconductor Bluetooth LE sniffer frames, capture length 65535)
```

First packet body:
```
1bff 4c00 0c0e 00db ...
```
`ff` = manufacturer-specific AD type, `4c00` = Apple Inc. — i.e. an iPhone
Continuity / AirDrop / Handoff advertisement, exactly what you'd expect on a
desk full of Apple gear.

To use with **Wireshark + the official extcap plugin**, see the `extcap/` folder
shipped with the Nordic sniffer zip — drop `nrf_sniffer_ble.py` into Wireshark's
extcap directory (`/Applications/Wireshark.app/Contents/MacOS/extcap` on macOS)
and `pip install -r extcap/requirements.txt`. The newer
`nrfutil ble-sniffer sniff` command is a drop-in CLI replacement that produces
the same PCAP format.

---

## Restoring connectivity firmware

The original `pc-ble-driver` connectivity firmware lives in the
[`pc-ble-driver` releases](https://github.com/NordicSemiconductor/pc-ble-driver/releases)
as `.hex`. Convert with the same UF2 family ID and drag back onto `UF2BOOT`.

---

*Captured: 2026-05-13. Tested on macOS 15 (Darwin 24.6), nrfutil 8.2.0,
nrf-sniffer 4.1.1, Makerdiary nRF52840-MDK-USB-Dongle bootloader 0.7.1.*
