# CTM Bridge — Controller Transport Mapping Bridge

Use the controllers paired to your **LG webOS TV** as if they were plugged
**straight into the gaming PC** — across your network, during a **Moonlight /
Sunshine** stream. Buttons, sticks, **and** the DualShock 4 / DualSense speaker,
headset and haptics. Even though a DS4/DS5 is **Bluetooth-paired to the TV**, the
PC sees a genuine **USB** controller.

## The short story

Lots of vibe-coding and lots of reverse engineering.

You can take a **DS5, DS4, Xbox controller, Steam Controller Puck** — and other
controllers — and have them appear on the remote PC as their **native USB
devices**, as if plugged in directly. It goes past input: DS4/DS5 **haptics and
audio** work too, reconstructed on the host by translation. The audio/haptics
signal is taken **straight from the USB composite device** (the controller's own
USB audio endpoints) — not synthesized.

I originally set out to write my **own virtual USB driver** (a Windows **UDE —
USB Device Emulation — `.sys`** driver). I stopped once I hit the driver-signing
costs and the real chance I couldn't get it WHQL-signed at all — so I switched
paths and built on top of an already-signed USB/IP client driver instead.

Huge thanks to **[usbip-win2](https://github.com/vadimgrn/usbip-win2)** by
**Vadym Hrynchyshyn (vadimgrn)** — its x64 driver is **WHQL-certified**, which is
exactly what made this path viable.

## How it works

1. The host agent answers a `CTM_DISCOVER` UDP broadcast on the LAN — **zero
   config**, no fixed IP (the TV learns the agent's address from the reply).
2. Moonlight starts a stream; the in-process bridge on the TV enumerates the
   paired controllers (DS4 / DS5 / Steam Puck / Xbox / generic HID). DS4/DS5 are
   typically **Bluetooth**-paired to the TV.
3. Each controller's HID is captured on the TV and tunneled over USB/IP. For a
   **Bluetooth** DS4/DS5 this means **reinterpreting the BT report format into the
   USB form the host expects** — the report IDs, headers, CRC and byte offsets
   differ between USB and Bluetooth — and translating host→controller output
   (lights, rumble, audio, haptics) back the other way.
4. `usbip-win2` re-exposes the result to Windows as a real **USB** device, so the
   game (and Windows audio) see a genuine controller — regardless of how it's
   actually connected to the TV.

## Design philosophy — blind driver · describing map · interpreting app

Three strictly separated layers:

- **The driver is blind.** `usbip-win2` and our transport only relay raw USB URBs
  and byte packets. They hold *zero* device-specific logic — they never know a
  DualSense from a DualShock, or USB from Bluetooth.
- **The map describes.** Each controller's byte shape — USB **and** Bluetooth HID
  report layouts, USB descriptors, audio/haptics blocks, route and volume bytes —
  is declared *statically* in a per-controller **map / profile**. A map is pure
  description; hardcoding `audio_block_id = 0x95` there is the map doing its job.
- **The app interprets.** The runtime reads the map and acts on it — including the
  USB↔Bluetooth reinterpretation. No device-specific decisions baked into C; a new
  controller quirk becomes a new map entry, not a new code path.

So adding a controller is mostly *reverse-engineer it and write a map*.

## Controllers

| Controller | Input | Audio / haptics | Notes |
|---|---|---|---|
| **DualSense (DS5)** | ✅ | ✅ translated | BT→USB reinterpreted; end-to-end at ~500 Hz; 48 kHz / 4-ch USB-ISO audio reconstructed |
| **DualShock 4 (DS4)** | ✅ | ⚠️ translated* | BT→USB reinterpreted; 32 kHz stereo; some route / rumble-vs-audio quirks still being ironed out |
| **Steam Controller (Puck)** | ✅ | — | full USB composite exposed |
| **Xbox** | ✅* | — | via the GIP map on the host side; native TV-side capture is limited by the webOS jail |
| **Generic HID / XInput** | ✅ | — | pass-through |

<sub>`*` work in progress — see each repo's docs for current state.</sub>

## Repositories

| Repo | Side | What it does |
|------|------|--------------|
| [**moonlight-tv**](https://github.com/CTM-Bridge/moonlight-tv) | TV (webOS) | Fork of [mariotaku/moonlight-tv](https://github.com/mariotaku/moonlight-tv) with CTM Bridge integrated in-process + an on-stream controller panel. |
| [**ctm-bridge-webos**](https://github.com/CTM-Bridge/ctm-bridge-webos) | TV (webOS) | Bridge core: HID capture, USB↔Bluetooth reinterpretation, per-controller translation, zero-config agent discovery, USB/IP client. |
| [**CTM-USBIP**](https://github.com/CTM-Bridge/CTM-USBIP) | Host (Windows / Linux) | Host agent: advertises on the LAN, accepts the bridged controller, re-exposes it via `usbip-win2`. |

## Credits

- **[usbip-win2](https://github.com/vadimgrn/usbip-win2)** by Vadym Hrynchyshyn — the WHQL-signed USB/IP client driver this builds on.
- **[Moonlight](https://moonlight-stream.org/)** & **[Sunshine](https://github.com/LizardByte/Sunshine)** — the streaming stack.
- **[mariotaku/moonlight-tv](https://github.com/mariotaku/moonlight-tv)** — the webOS Moonlight client this forks.

## Disclaimer

This is an independent, non-commercial hobby project.

All reverse engineering was done **cleanly**, using only **publicly available
information** (public wikis, open-source code, public documentation) and the
author's **own observation of hardware they own** — USB / Bluetooth captures of
the author's own controllers. **No protected, proprietary, confidential, NDA'd,
or leaked material was used, and no official driver, SDK, or firmware was
decompiled or disassembled.**

Not affiliated with, endorsed by, or sponsored by Sony, Microsoft, Valve, LG, or
NVIDIA. *DualShock*, *DualSense*, *Xbox*, *Steam Controller*, *webOS*,
*Moonlight*, *Sunshine* and other names are trademarks of their respective
owners, used here only to describe compatibility.
