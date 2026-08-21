# Samsung Galaxy Book6 Ultra (NP960UJG-KG2FR) on Linux

## System

- Model: Samsung Galaxy Book6 Ultra (NP960UJG-KG2FR | PVAL)
- CPU: Intel Core Ultra X7 358H
- iGPU: Intel Arc B390
- RAM: 32 GB
- Storage: 1 TB NVMe SSD
- Operating system: Aurora 44 (based on Fedora 44 Kinoite)
- Kernel: 7.1.5-200.fc44.x86_64 (64-bit)
- Graphics platform: Wayland

## Display

### Internal OLED panel

- Resolution: 2880x1800
- Refresh rates: 60 Hz and 120 Hz
- HDR and wide-gamut support are present in the panel, but KDE Plasma currently reports both as unavailable.
- The panel EDID advertises DisplayID 2.0 with an encapsulated CTA-861 block containing BT.2020 RGB and SMPTE ST 2084/PQ metadata, with approximately 1100 nits peak brightness.
- Installed `libdisplay-info 0.3.0` does not search CTA blocks nested in DisplayID 2.0. KWin therefore receives neither HDR nor wide-gamut capability.
- The upstream parser fix is included from `libdisplay-info 0.4.0`; Aurora/Fedora 44 still provides `0.3.0` and needs an update or backport.
- Local status: `kscreen-doctor` reports `HDR: incapable` and `Wide Color Gamut: incapable` for `eDP-1`.
- Workaround: add `KWIN_FORCE_ASSUME_HDR_SUPPORT=1` to `/etc/environment` to expose the HDR option in KDE Display Configuration.
  - HDR content displays correctly, but SDR content is dimmer than expected.
- References: [KWin HDR EDID override](https://invent.kde.org/plasma/kwin/-/merge_requests/7337), [libdisplay-info nested CTA fix](https://chromium.googlesource.com/external/gitlab.freedesktop.org/emersion/libdisplay-info/+/73ec53d800275cce4b8a4af6af03ea2aac9912fd), [Samsung panel measurements](https://www.notebookcheck.com/Samsung-Galaxy-Book6-Ultra-im-Test-Beeindruckender-Multimedia-Laptop-mit-tollem-OLED-und-RTX-5070.1243968.0.html)

### Adaptive Sync and Panel Replay

- If Adaptive Sync is enabled, use kernel parameter `xe.enable_panel_replay=0`.
- Without it, the internal screen may show artifacts, become unusable intermittently, or trigger a full system crash under internal display load.
- Enabling this parameter seems to slightly reduce battery life.
- If battery life is the priority, disabling Adaptive Sync entirely may be preferable. Occasional artifacts can still occur, especially after multiple sleep/wake cycles with Electron applications.
- Forcing active sync mode instead of automatic mode feels sluggish, with persistent low-FPS behavior and a less responsive cursor/touchpad.

### Display status with ambient-light firmware installed

```text
# kscreen-doctor --outputs
Output: 1 eDP-1 bc38b1e1-806f-4fed-9eb9-5dedf06de865
enabled
connected
priority 1
Panel
replication source:0
Modes:  1:2880x1800@120.00!  2:2880x1800@60.00*  3:1600x1200@59.87  4:1600x1200@119.82  5:1280x1024@59.90  6:1280x1024@119.83  7:1024x768@59.92  8:1024x768@119.80  9:2560x1600@59.99  10:2560x1600@119.93  11:1920x1200@59.88  12:1920x1200@119.90  13:1280x800@59.81  14:1280x800@119.85  15:2880x1620@59.96  16:2880x1620@119.95  17:2560x1440@59.96  18:2560x1440@119.95  19:1920x1080@59.96  20:1920x1080@119.93  21:1600x900@59.95  22:1600x900@119.95  23:1368x768@59.88  24:1368x768@119.83  25:1280x720@59.85  26:1280x720@119.86
Custom modes: None
Geometry: 0,0 1920x1200
Scale: 1.5
Rotation: 1
Overscan: 0
Vrr: Never
RgbRange: Automatic
HDR: disabled
Wide Color Gamut: disabled
ICC profile: none
Color profile source: sRGB
Color power preference: prefer efficiency and performance
Brightness control: supported, set to 13% and dimming to 100%
Color resolution: automatic (10), range: [6; 12] bits per color
Allow EDR: unsupported
Sharpness control: supported, set to 0%
Automatic brightness: supported, enabled
Auto Rotate Policy: incapable
Adaptive backlight modulation: unsupported
```

## Audio

- Fedora 44+ and current Arch Linux packages include the required Cirrus Logic CS35L57 firmware for this laptop (`144dca0a`). Aurora 44 therefore needs no manual extraction or firmware-path configuration.
- Audio quality should already be good without additional firmware work. The packaged firmware restores the built-in speakers' intended character without Dolby processing or other artificial enhancements.
- Ubuntu and Debian packages checked on 22 August 2026 did not yet contain the exact `144dca0a` files; other distributions may require the fallback procedure until their packages update.
- Fallback extraction procedure: [AUDIO-CIRRUS-FIRMWARE-EXTRACTION.md](./AUDIO-CIRRUS-FIRMWARE-EXTRACTION.md)
- Related issue: https://github.com/antoinecellerier/speaker-tuning-to-easyeffects/issues/27

## Storage

- LUKS encryption limits NVMe throughput to about 4 GB/s instead of approximately 7 GB/s unencrypted. Daily usage is generally not noticeably affected.
- SSD appears to support OPAL. Correct configuration could offload encryption to the SSD controller and provide near-native performance.
- On Windows 11, performance appears to remain near maximum with BitLocker enabled.

## Sensors and Input

### Ambient light sensor

- KDE does not currently detect the ambient light sensor, so automatic brightness adjustment is unavailable.
- It can work after installing Samsung-specific Panther Lake ISH firmware.
- Generic `ish_ptl.bin` is rejected with `ISH loader: cmd 2 failed 10`; no ISH HID/IIO sensor is enumerated without the Samsung-specific firmware.
- Extraction and installation procedure: [AMBIENT-LIGHT-SENSOR-FIRMWARE-EXTRACTION.md](./AMBIENT-LIGHT-SENSOR-FIRMWARE-EXTRACTION.md)
- Procedure covers Ubuntu, Fedora non-Atomic, and Fedora Atomic systems.
- With firmware installed, `kscreen-doctor` reports `Automatic brightness: supported, enabled`.

### Fingerprint reader

- Does not work, with no visible KDE integration.
- Can be made to work by patching fprintd, for example: https://github.com/TenSeventy7/libfprint-egismoc-sdcp/issues/9

### Touchpad and keyboard shortcuts

- Fn shortcuts not working:
  - Fn+F1: Settings
  - Fn+F10: Microphone mute toggle
  - Fn+F11: Camera toggle
- `dmesg` shows unknown ACPI notification events.
- Related issue: https://github.com/joshuagrisham/samsung-galaxybook-extras/issues/108

## Camera

- Webcam is not recognized yet.
- A project has managed to make it work, so an additional device ID declaration in the kernel or driver may be sufficient.
- Related issue: https://github.com/intel/ipu7-drivers/issues/73#issuecomment-4881775585

## Thermal Management

- `quiet` and `max_fan` platform profiles are missing.
- Available profiles through `/sys/firmware/acpi/platform_profile_choices`:

```text
# cat /sys/firmware/acpi/platform_profile_choices
low-power balanced performance
```

- `low-power` corresponds to Silent, `balanced` to Optimized, and `performance` to High performance. The `quiet` profile (Silent), which is available on Windows 11, is missing. The `max_fan` profile (Maximum fan speed) is also unavailable because firmware mode `0x17` is not mapped in `profile_performance_modes`.
Also, A potentially dangerous bug prevents the fans from activating when the `balanced` profile is selected, especially when switching from `low-power` (precedence) or cold-start. This can cause overheating during intensive CPU/GPU use and make the entire chassis dangerously hot. This problem does not occur on Windows 11, where the fans activate more frequently under the same usage conditions. Switching to performance mode before returning to balanced mode reactivates the fans. This resembles [a bug already reported in the samsung-galaxybook-extras repository](https://github.com/joshuagrisham/samsung-galaxybook-extras/issues/95#issuecomment-5307708030).

## Power Management

- Battery life is good and likely comparable to Windows 11 in practical use; no strict side-by-side benchmark performed.
- More than 12 hours is achievable for software development workloads at 20% brightness using Tuned Power Save.
- A lighter, more optimized IDE than Visual Studio Code may exceed 15 hours under similar conditions.
- Sleep mode (`s2idle`) has relatively low drain, approximately 0.38% per hour, though it remains less efficient than a MacBook.

## Overall Status

- Most hardware works correctly overall.
