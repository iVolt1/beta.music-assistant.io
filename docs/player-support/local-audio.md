---
title: Local Audio Out Player Provider
description: Play audio through locally attached soundcards — USB DACs, built-in audio, HDMI output, and multi-channel surround cards — directly from the Music Assistant host machine.
---

# Local Audio Out

The Local Audio Out provider exposes soundcards physically attached to the Music Assistant host as players. USB DACs, built-in audio chips, HDMI audio output, and multi-channel surround cards (5.1, 7.1) are all supported. Each output appears as an independent MA player with its own volume control, and players can be grouped for synchronized multi-room playback via the Sendspin provider.

:::note
This provider is currently in **alpha**. It is functional but may have rough edges. Feedback and bug reports are welcome via the [Music Assistant GitHub discussions](https://github.com/orgs/music-assistant/discussions).
:::

## Requirements

- Music Assistant running on **Linux** (Home Assistant OS recommended) or **macOS**
- For the PulseAudio/PipeWire backend *(Linux)*: PulseAudio or PipeWire with a PulseAudio compatibility layer must be running on the host. On Home Assistant OS this is provided automatically by the MA audio addon
- For the ALSA direct backend *(Linux)*: no additional software required
- The **Sendspin** provider must be enabled — Local Audio Out depends on it for synchronized playback

## Features

- Automatic device discovery — no manual configuration of device names or addresses
- Synchronized multi-room playback when players are grouped
- Per-player hardware volume control on the PulseAudio/PipeWire backend, with an audio taper curve for consistent dB-per-step behaviour across the full slider range
- Native sample rate and bit depth negotiation (16, 24, or 32-bit) on the PulseAudio/PipeWire backend — no unnecessary resampling
- Self-managed multi-channel remap-sink topology for 5.1 and 7.1 surround cards — no external addon or manual `pactl` configuration required
- PipeWire compatible

## Setup

1. Navigate to **Settings → Providers** and click **Add a new provider**
2. Select **Local Audio Out** from the player providers list
3. *(Linux only)* Choose an **Audio backend** — see [Settings](#settings) below. The default **Auto** setting works for most installations
4. Click **Save**. The provider will discover all available output devices and register them as MA players

Discovered players appear immediately in the MA player list. Each soundcard output — including individual zone sinks on multi-channel cards — is registered as its own player.

## Settings

In addition to the [Individual Player Settings](/settings/individual-player), Local Audio Out has one provider-level setting on Linux:

### Audio Backend *(Linux only)*

| Option | Description |
|--------|-------------|
| **Auto** *(default)* | Uses PulseAudio/PipeWire if available, falls back to ALSA direct |
| **PulseAudio / PipeWire** | Forces PulseAudio/PipeWire. Fails at startup if PulseAudio is not running |
| **ALSA (direct)** | Direct hardware access via ALSA `hw:` nodes, bypassing PulseAudio/PipeWire |

The PulseAudio/PipeWire backend is recommended for most users — it supports hardware volume control, multi-channel remap-sink topology, and the widest range of devices including USB DACs and virtual sinks. Use ALSA direct only if a device cannot be used correctly through PulseAudio/PipeWire.

## Multi-Channel Sound Cards (5.1 / 7.1)

On the PulseAudio/PipeWire backend, multi-channel surround cards are automatically split into independent stereo zone players — no addon or manual setup required.

For every surround card detected, the provider creates:

- **`<card>_front_stereo`** — front left/right channel pair
- **`<card>_rear_stereo`** — rear left/right channel pair (if present)
- **`<card>_side_stereo`** — side left/right channel pair (if present, 7.1 cards)
- **`<card>_center_sub`** — front centre + LFE (subwoofer) channel pair (if present)
- **`<card>_multichannel_stereo`** — full-channel passthrough sink (6+ channel cards), equivalent to an AVR's "Multi Channel Stereo" or "All Channel Stereo" mode — plays the same audio to all outputs simultaneously

Each zone sink has its own independent hardware volume in MA and does not affect the volume of any other zone. The raw multi-channel master sink is not registered as a separate player — it is fully covered by the zone sinks.

The topology is created automatically on every provider startup and cleaned up on provider stop. If a zone sink already exists from a previous run it is left untouched (idempotent). Card names are derived from the ALSA card name (e.g. "Creative X-Fi" becomes `Creative_X_Fi`).

## Volume Control

Volume and mute are applied differently depending on the backend:

**PulseAudio/PipeWire backend**: Volume is applied as native PulseAudio sink hardware volume via `libpulse`. Each player's volume is fully independent and does not affect any other sink. The MA volume slider uses an exponential audio taper curve — each step of the slider changes loudness by the same number of dB, unlike a linear mapping where the bottom of the slider is overly sensitive. Volume level is restored after a provider restart; mute state is intentionally **not** restored (players always start unmuted to prevent silent-startup surprises).

**ALSA direct backend**: Volume is applied in software by scaling PCM samples before they are written to the device. Hardware mixer levels (Master, PCM) are not managed by MA — set them to 100% once using `alsamixer` and persist with `sudo alsactl store` so they survive reboots.

**macOS**: Same as ALSA direct — software volume only.

## Known Issues / Notes

- **USB device plug/unplug causes a full provider reload** on Linux. All active playback on all local audio players stops briefly during the reload (typically under a second). The new or removed device is reflected automatically and playback resumes normally on the next play request
- **PA sinks stay in RUNNING state** while the provider is active. This is intentional — streams are pre-opened at startup to eliminate cold-start sync offset when multiple players start together in a sync group. Sinks return to IDLE when the provider is disabled or MA is stopped
- **ALSA direct backend**: PortAudio enumerates only real hardware `hw:` nodes. Virtual PCM plugins (`sysdefault`, `dmix`, `surround*`, etc.) are excluded. If a device is held exclusively by another process it is silently skipped at enumeration time
- **Sample rate and bit depth** on the PulseAudio/PipeWire backend are determined by the PA daemon configuration and the sink's hardware capabilities — they are not configurable per-player in MA. On Home Assistant OS these are set in the MA audio addon configuration
- **Silent output after a PA daemon restart**: On some ALSA cards (notably the Creative X-Fi on kernel 6.12.x) a driver stall can cause silent output after a PA daemon restart or sample rate change. Reloading the provider via **Settings → Providers → Local Audio Out → Reload** resolves this. As a manual workaround: `pactl suspend-sink <master_sink_name> true && sleep 1 && pactl suspend-sink <master_sink_name> false`
- **macOS support** is intended for developers running MA locally. It is not supported on Home Assistant OS
- If a provider reload is needed after adding or removing PA sinks or ALSA devices, use **Settings → Providers → Local Audio Out → Reload**
