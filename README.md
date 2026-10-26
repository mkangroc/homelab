# Homelab Network & Service Infrastructure

A home network and self-hosted service setup I designed, built, and run on repurposed
hardware. This repository documents the **architecture and the reasoning behind it**.

Everything here runs continuously: a home network with its own firewall, several
self-hosted services, and remote access from outside the house.

---

## Architecture

![Network architecture](docs/architecture.svg)

**How it fits together:** my internet connection comes into an **OPNsense** box I built
from an old desktop, which became the router and firewall for the whole network. The
ISP's own gateway was demoted to access-point mode — it only provides Wi-Fi and switch
ports now, it doesn't route. Behind that, devices sit on separate network interfaces
with their own rules. A room on the far side of the house is reached using the coax
cable already in the walls, and the router in that room can send chosen devices through
a VPN.

The reasoning behind each of those choices is in
[`docs/design-decisions.md`](docs/design-decisions.md).

## Stack

| Service | Role | Notes |
|---|---|---|
| **OPNsense** | Router / firewall | Runs DHCP and DNS for my whole network, with blocklists applied |
| **Tailscale** | Remote access | Lets me reach my services from outside the house without opening any ports to the internet |
| **Immich** | Photo & video backup | Self-hosted replacement for a paid cloud photo service; backs up automatically from my phone |
| **Open WebUI** | Local AI chat | A chat interface for language models running on my own hardware |
| **GL.iNet + NordVPN** | Separate network segment | Sends the devices I choose through a VPN while everything else goes out directly |

## Design highlights

1. **Separate networks instead of one flat network.** The main network and remote-access
   devices each get their own interface with their own rules,
   so a device on one isn't automatically able to reach the others.
3. **Local DNS with blocklists.** Name lookups happen on my router, so no outside company
   gets a list of every site my network visits, and the blocklists apply to every device
   without installing anything on them.
4. **Reusing the coax already in the walls.** The room I wanted on the network had coax
   run to it but no network cable, so I used MoCA adapters — ethernet over coax — instead
   of running new cable through a finished wall.
5. **Per-device VPN choice.** Devices I select route through a VPN; everything else goes
   out directly.

## Repository layout

```
.
├── README.md
├── docs/
│   ├── architecture.svg      network diagram
│   └── design-decisions.md   why each choice was made
└── examples/                 sanitized Compose templates
    ├── open-webui/
    ├── vaultwarden/
    └── immich/
```
