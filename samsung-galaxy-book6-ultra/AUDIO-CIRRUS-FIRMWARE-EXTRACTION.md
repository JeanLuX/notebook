# Cirrus Audio Firmware for the Samsung Galaxy Book6 Ultra on Linux

This procedure extracts the firmware for the six Cirrus Logic CS35L57
amplifiers from Samsung's official Windows driver, prepares the names expected
by Linux, and installs it.

> **Fedora 44+ and current Arch Linux — no manual extraction needed (since 11
> August 2026).** Upstream `linux-firmware` now includes the required files for
> SSID `144dca0a`. Fedora ships them in [build
> 3077583](https://koji.fedoraproject.org/koji/buildinfo?buildID=3077583);
> Arch ships them in the `linux-firmware-cirrus` package.
>
> On Fedora 44+ or current Arch Linux, update the normal firmware package and do
> nothing else: do not extract the Samsung driver, copy files manually, add
> `firmware_class.path`, or regenerate an initramfs for this firmware. Sound
> quality should already be good without further action. No Fedora- or
> Arch-specific installation steps are needed.
>
> **This is not yet true for every distribution.** The Ubuntu and Debian package
> file lists checked on 22 August 2026 do not contain the exact `144dca0a` files;
> openSUSE and Gentoo were not confirmed. Keep this procedure as a fallback
> unless the installed package contains
> `cirrus/cs35l57-b2-dsp1-misc-144dca0a.wmfw` and its six `.bin` files.

## 1. Scope and Limitations

### Supported Hardware

**This procedure is intended exclusively for the following configuration:**

| Component | Expected value |
|---|---|
| Ordinateur | **Samsung Galaxy Book6 Ultra** |
| PCI audio controller | **Intel `[8086:e428]`** |
| PCI subsystem | **Samsung `[144d:ca0a]`** |
| SSID used by Linux | **`144dca0a`** |
| Amplificateurs | **6 x Cirrus Logic CS35L57 Rev B2** |
| Bus | **SoundWire** |
| Adresses SoundWire | **`l1u0`, `l1u1`, `l1u2`, `l2u3`, `l2u4`, `l2u5`** |

The Windows driver and some directory names use the generic name `CS35L56`.
However, the hardware identifiers specify `PART_3557`, so the Linux driver
builds the `cs35l57-b2` prefix.

### Hardware Warning

**The firmware and coefficients for the Cirrus amplifiers are specific to the
laptop model, speakers, and their wiring. Never use files from another computer,
even one that appears similar. Incorrect voltage, current, or impedance
parameters can damage the speakers.**

**These instructions are provided without warranty. You follow them at your own
risk and with full knowledge of the possible consequences. The author cannot be
held responsible for damage to the device, speakers, or any other consequence
resulting from their use.**

### Software Scope

This procedure installs only the CS35L57 DSP firmware and coefficients. Full
audio functionality also requires compatible SOF firmware, an ALSA topology,
and a UCM configuration.

**Do not treat the Windows APO, Dolby, Fortemedia, or CS42L45 firmware
components also present in the Samsung archive as CS35L57 firmware.**

## 2. Source and Prerequisites

### Download the Official Driver

The driver is available from the
[Samsung Galaxy Books Download Center](https://www.samsung.com/global/galaxybooks-downloadcenter/).

1. **Select the exact Galaxy Book6 Ultra model.**
2. **Open the `Sound` section** of the driver list.
3. **Locate the driver named `Cirrus - Sound Driver`.**
4. **Download its ZIP archive** to the working directory.

**Do not use the Cirrus driver provided for another model.** The archive name
may change between versions; in the examples below, the file is named
`BASW-A4285A24_1063.zip`.

### Samsung Archive

The archive contains the Cirrus driver, its DSP firmware, and the amplifier
tuning files. Names and subdirectories may change between versions; the
procedure searches for them after extraction.

### Required Tools

The following commands must be available:

```text
unzip  install  ln  find  lspci  journalctl
```

`dracut` is used only on distributions that require it.

## 3. Check the System

### PCI Controller and SSID

```bash
lspci -v -nn | grep -A2 -i audio
```

The output should include, among other information:

```text
Subsystem: Samsung Electronics Co Ltd Device [144d:ca0a]
```

**Do not continue if the subsystem is not `[144d:ca0a]`.**

### SoundWire Amplifiers

```bash
journalctl -k -b | grep -iE 'cirrus|cs35l'
```

The output must identify **six `CS35L57 Rev B2`** amplifiers at the addresses
**`l1u0`, `l1u1`, `l1u2`, `l2u3`, `l2u4`, and `l2u5`**.

**The following errors generally confirm that the hardware is detected but its
specific files are missing:**

```text
FIRMWARE_MISSING
Calibration disabled due to missing firmware controls
Can't read tuning IDs
```

## 4. Extract and Check the Samsung Driver

### Define the Working Paths

The ZIP is searched for in the current directory; extraction and preparation are
performed under `${TMPDIR:-/tmp}`:

```bash
ZIP=./BASW-A4285A24_1063.zip
WORK_DIR="${TMPDIR:-/tmp}"
DRIVER_DIR="$WORK_DIR/BASW-A4285A24_1063"
OUT="$WORK_DIR/samsung-np960ujg-cs35l57-revb2-firmware"
```

### Check and Extract the ZIP

First check the archive's integrity:

```bash
unzip -tq "$ZIP"
```

Extract the archive into the working directory:

```bash
mkdir -p "$DRIVER_DIR"
unzip -oq "$ZIP" -d "$DRIVER_DIR"
```

The `-o` option replaces files from a previous extraction in this working
directory without prompting.

### Find the Extracted Files

```bash
CS_DIR=$(find "$DRIVER_DIR" -type d -name CS -print -quit)
SAMSUNG_DIR="$CS_DIR/XU_Ext/Samsung"
FW=$(find "$SAMSUNG_DIR/fw" -type f -name '*.wmfw' -print -quit)
TN_DIR=$(find "$SAMSUNG_DIR/tn" -type d -iname 'CA0A' -print -quit)
```

**The DSP firmware is the first `.wmfw` file** found in the Samsung directory.
If multiple files are present, select the one for the CS35L56/CS35L57 firmware
used by this laptop.

The six coefficient files correspond to the following SoundWire addresses:

| Role | Address |
|---|---|
| Left woofer 1 | `l1u0` |
| Left woofer 2 | `l1u1` |
| Left tweeter | `l1u2` |
| Right woofer 1 | `l2u3` |
| Right woofer 2 | `l2u4` |
| Right tweeter | `l2u5` |

## 5. Prepare the Names Expected by Linux

### Naming Convention

The driver requests files relative to the firmware root, under `cirrus/`, using
one of the following forms:

```text
cs35l57-b2-dsp1-misc-144dca0a-lXuY.wmfw
cs35l57-b2-dsp1-misc-144dca0a-lXuY.bin
cs35l57-b2-dsp1-misc-144dca0a-spkid0-lXuY.wmfw
cs35l57-b2-dsp1-misc-144dca0a-spkid0-lXuY.bin
```

The `spkid0` form depends on the ACPI properties exposed by the machine. The
Windows package sets `SpeakerSetValue=0`; both naming forms are therefore
prepared to cover the two names Linux may construct.

### Create the Installation-Ready Directory

```bash
BASE=cs35l57-b2-dsp1-misc-144dca0a

install -d -m 0755 "$OUT"

install -m 0644 "$FW" "$OUT/$BASE-l1u0.wmfw"

for address in l1u0 l1u1 l1u2 l2u3 l2u4 l2u5; do
  source=$(find "$TN_DIR" -type f -iname "*${address}.bin" -print -quit)
  install -m 0644 "$source" "$OUT/$BASE-$address.bin"
  if [ "$address" != l1u0 ]; then
    ln -sfn "$BASE-l1u0.wmfw" "$OUT/$BASE-$address.wmfw"
  fi
  ln -sfn "$BASE-l1u0.wmfw" "$OUT/$BASE-spkid0-$address.wmfw"
  ln -sfn "$BASE-$address.bin" "$OUT/$BASE-spkid0-$address.bin"
done
```

The directory contains **24 entries**, but only **7 payloads**: one shared
`.wmfw` firmware file and six `.bin` coefficients. The other 17 entries are
internal symbolic links and occupy virtually no space.


### Check the Result

```bash
for file in "$OUT"/*; do
  test -e "$file" || printf 'Broken link: %s\n' "$file"
done
```

**This command should produce no output.**

You can then view all firmware files copied to the working directory:


```bash
xdg-open "$WORK_DIR/samsung-np960ujg-cs35l57-revb2-firmware"
```

## 6. Install the Firmware

### How Search Paths Work

The kernel normally searches for firmware under `/lib/firmware`, which generally
points to `/usr/lib/firmware` on usr-merged systems.

The **`firmware_class.path`** parameter adds a priority path without disabling
the standard paths. If a file does not exist in the custom path, the search
continues under `/lib/firmware`. If the same name exists in both locations, the
file in the custom path takes priority.

### Ubuntu 26.04 LTS

**Ubuntu 26.04 LTS uses Dracut by default.** Install into the standard path:

```bash
sudo install -d -m 0755 /usr/lib/firmware/cirrus
sudo cp -a "$OUT"/. /usr/lib/firmware/cirrus/
sudo dracut --regenerate-all --force
```

**`firmware_class.path` is not required.** Regeneration allows existing initramfs
images to include the new installation.

### Power Off and Restart

**After copying the firmware, fully power off the computer and start it again
manually** to completely reset the amplifiers and the SoundWire bus.

```bash
sudo systemctl poweroff
```

**Wait for the computer to power off completely**, then press the power button
to start it.

## 7. Check After a Cold Boot

### Firmware Loading

```bash
journalctl -k -b | grep -iE 'cirrus|cs35l|calibration'
```

A successful initialization should mention, for each of the six amplifiers, the
loading of a `.wmfw` file and a `.bin` file, followed by:

```text
Calibration applied
```

The presence of **`Calibration applied`** confirms that calibration was applied.

The names logged should begin with:

```text
cirrus/cs35l57-b2-dsp1-misc-144dca0a-
```

### Absence of the Initial Errors

```bash
journalctl -k -b | grep -iE \
  'FIRMWARE_MISSING|missing firmware controls|Can.t read tuning IDs'
```

**This command should produce no output.**

The messages `supply ... not found, using dummy regulator` are not, by
themselves, errors. They also appear in the successful initialization example
in the kernel documentation.

## 8. Update or Disable the Firmware

### Update

**For a new official version of the Samsung driver:**

1. Replace the ZIP in the working directory.
2. Repeat sections 4 and 5 to find and prepare the new files.
3. Copy the new contents to the destination used by the system.
4. Regenerate the initramfs on Ubuntu.
5. Fully power off, restart, and check the kernel log.

### Disable on Ubuntu

Move only this machine's files out of the active path:

```bash
sudo install -d -m 0755 /usr/lib/firmware/cirrus.disabled
sudo mv /usr/lib/firmware/cirrus/cs35l57-b2-dsp1-misc-144dca0a-* \
  /usr/lib/firmware/cirrus.disabled/
sudo dracut --regenerate-all --force
sudo systemctl poweroff
```

Wait for the computer to power off completely, then start it manually.

## 9. Troubleshooting

### `FIRMWARE_MISSING` Persists

Check the **`cs35l57-b2`** prefix, the **`144dca0a`** SSID, the SoundWire address,
and the `cirrus` subdirectory.

### The `.wmfw` Loads but the `.bin` Does Not

Check that the `.bin` file associated with each address exists and that the
`spkid0` links are not broken. A `.bin` from another address or another machine
is not interchangeable.

### Calibration Still Fails

Missing, empty, corrupted, or no-longer-matching EFI calibration data can cause
a separate error even when the firmware has loaded. Review the driver's full
messages without overly restrictive filtering.

### The Firmware Loads but Audio Remains Incomplete

Check the SOF firmware, ALSA topology, UCM configuration, PipeWire, and active
audio profile separately. These components are not provided by this procedure.

## References

- [Galaxy Books Download Center de Samsung](https://www.samsung.com/global/galaxybooks-downloadcenter/)
- [Linux driver for the CS35L54/56/57/63](https://docs.kernel.org/sound/codecs/cs35l56.html)
- [Linux firmware search paths](https://docs.kernel.org/driver-api/firmware/fw_search_path.html)
- [Ubuntu 26.04 LTS notes: transition to Dracut](https://documentation.ubuntu.com/release-notes/26.04/summary-for-lts-users/)
- [Upstream `linux-firmware` entries for SSID `144dca0a`](https://kernel.googlesource.com/pub/scm/linux/kernel/git/firmware/linux-firmware/%2B/refs/heads/main/WHENCE#9201)
- [Arch Linux `linux-firmware-cirrus` file list](https://archlinux.org/packages/core/any/linux-firmware-cirrus/files/)
- [Ubuntu `linux-firmware` file list](https://packages.ubuntu.com/noble-updates/armhf/linux-firmware/filelist)
- [Debian sid `firmware-cirrus` file list](https://packages.debian.org/sid/all/firmware-cirrus/filelist)
