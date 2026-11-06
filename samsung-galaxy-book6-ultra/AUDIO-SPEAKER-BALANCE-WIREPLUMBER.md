# Permanent Speaker Balance Correction for the Samsung Galaxy Book6 Ultra on Linux

The built-in speakers of the Samsung Galaxy Book6 Ultra (NP960UJG) do not play
at the same level on the left and right channels: the sound is noticeably
unbalanced toward the left side. This document describes a permanent,
system-wide correction applied in WirePlumber so the balance no longer has to
be set manually in PulseAudio/PipeWire after every session or reboot.

Related kernel bug report:
[bugzilla.kernel.org #221964](https://bugzilla.kernel.org/show_bug.cgi?id=221964)

## 1. Approach

PipeWire's session manager, WirePlumber, supports per-node SPA filter graphs.
A small `linear` gain filter is attached directly to the existing **Speaker**
output sink and attenuates only the left channel:

- Left channel: linear gain `0.658503`
- Right channel: linear gain `1.0`

Because the filter is attached to the sink itself, the sink keeps its name and
its volume control in KDE, and the correction survives reboots, session
changes, and volume changes. The configuration is installed globally under
`/etc/wireplumber`, so every user of the machine benefits from it.

On the cubic volume scale used by `pactl` and PulseAudio, a volume of 87 %
corresponds to a linear amplitude of `0.87³ ≈ 0.658503`.

> **The gain value is measured on this specific unit.** Physical speaker
> tolerances vary between individual laptops. Use the value below as a
> starting point and adjust it to taste for your own unit.

## 2. Requirements

- Samsung Galaxy Book6 Ultra with working built-in speakers (see
  [AUDIO-CIRRUS-FIRMWARE-EXTRACTION.md](./AUDIO-CIRRUS-FIRMWARE-EXTRACTION.md)
  if the Cirrus Logic firmware is missing).
- WirePlumber 0.5 or later as the PipeWire session manager (default on
  current Fedora, Aurora, Ubuntu, and Arch Linux).
- Tested on Aurora 44 (based on Fedora 44 Kinoite), KDE Plasma, Wayland.

## 3. Identify the Speaker Sink Name

The filter matches the sink by its WirePlumber node name. Check the name on
your system before installing the configuration:

```bash
pactl list short sinks
```

On this model, the internal speaker sink is:

```text
alsa_output.pci-0000_00_1f.3-platform-sof_sdw.HiFi__Speaker__sink
```

If your system shows a different name, use yours in the configuration file
below.

## 4. Create the Global Configuration

Run the following block in a Bash or Zsh terminal:

```bash
sudo install -d -m 0755 /etc/wireplumber/wireplumber.conf.d

sudo tee /etc/wireplumber/wireplumber.conf.d/51-speaker-balance-samsung-np960.conf > /dev/null <<'EOF'
# Samsung NP960: permanent balance correction for the built-in speakers.
# Equivalent to a PulseAudio volume ratio of 87% left / 100% right.

node.filter-graph.rules = [
  {
    matches = [
      {
        node.name = "alsa_output.pci-0000_00_1f.3-platform-sof_sdw.HiFi__Speaker__sink"
      }
    ]

    actions = {
      create-filter-graph = [
        {
          nodes = [
            {
              type = builtin
              name = left
              label = linear
              control = { Mult = 0.658503 Add = 0.0 }
            }
            {
              type = builtin
              name = right
              label = linear
              control = { Mult = 1.0 Add = 0.0 }
            }
          ]

          inputs = [ "left:In" "right:In" ]
          outputs = [ "left:Out" "right:Out" ]
        }
      ]
    }
  }
]
EOF

sudo chmod 0644 /etc/wireplumber/wireplumber.conf.d/51-speaker-balance-samsung-np960.conf
```

WirePlumber reads drop-in files from this directory for all users. The filter
is applied to the existing **Speaker** sink, which keeps its name and its
volume control in KDE. See
[WirePlumber configuration locations](https://pipewire.pages.freedesktop.org/wireplumber/daemon/locations.html).

## 5. Activate the Configuration

In your session, **without `sudo`**:

```bash
systemctl --user restart wireplumber.service
```

This briefly interrupts audio playback. The `--user` service also reads the
global files under `/etc`. Other accounts pick up the configuration at the
next start of their audio session.

## References

- [Kernel bug report #221964 — unbalanced speaker output](https://bugzilla.kernel.org/show_bug.cgi?id=221964)
- [WirePlumber configuration file locations](https://pipewire.pages.freedesktop.org/wireplumber/daemon/locations.html)
- [WirePlumber config fragments (filter-graph)](https://pipewire.pages.freedesktop.org/wireplumber/daemon/configuration/fragments.html)
- [PulseAudio volume scale (cubic)](https://freedesktop.org/software/pulseaudio/doxygen/volume.html)
- [SPA filter-graph plugin](https://docs.pipewire.org/page_module_filter_chain.html)
