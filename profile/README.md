# CTM Bridge — Controller Transport Mapping Bridge

**CTM Bridge** bridges game controllers paired to an **LG webOS TV** through to a
**Windows / Linux host** over **USB/IP**, so they work in games streamed with
**Moonlight / Sunshine** — the host sees each controller as its *native* USB
device (DualShock 4, DualSense, Steam Controller, Xbox, generic XInput) instead
of going through SDL → ViGEmBus / uinput.

The bridge runs **in-process inside the Moonlight TV client**: when a stream
starts, controllers paired to the TV are captured, translated, and tunneled to a
host-side agent that re-exposes them via USB/IP.

## Repositories

| Repo | Side | What it does |
|------|------|--------------|
| [**moonlight-tv**](https://github.com/CTM-Bridge/moonlight-tv) | TV (webOS) | Fork of [mariotaku/moonlight-tv](https://github.com/mariotaku/moonlight-tv) with CTM Bridge integrated in-process + an on-stream controller panel. |
| [**ctm-bridge-webos**](https://github.com/CTM-Bridge/ctm-bridge-webos) | TV (webOS) | The bridge core: HID capture, per-controller translation/patching, zero-config agent discovery, USB/IP client. |
| [**CTM-USBIP**](https://github.com/CTM-Bridge/CTM-USBIP) | Host (Windows / Linux) | The host agent: advertises itself on the LAN, accepts the bridged controller, and re-exposes it to games via USB/IP. |

## How it works

1. The host agent listens on the LAN and answers a `CTM_DISCOVER` UDP broadcast —
   **zero-config**, no fixed IP (the TV learns the agent's address from the reply).
2. On the TV, Moonlight starts a stream and the embedded bridge enumerates paired
   controllers (DS4 / DS5 / Steam Controller / Xbox / generic HID).
3. Each controller's HID stream is captured on the TV, translated per a device
   map, and tunneled to the agent.
4. The agent re-exposes it to the host OS as a real USB device via USB/IP, so the
   game sees a native controller.

## Status

Work in progress. DualSense bridges end-to-end; DualShock 4, Steam Controller,
and generic HID are supported, each with per-controller settings (audio routing,
volumes, latency, haptics) surfaced in the on-stream panel. See each repository
for build instructions and current state.
