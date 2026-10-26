# Design Decisions

Why the network is built the way it is, and what went wrong while building it.

---

## Constraints I was working within

- **No new cabling.** The far room had coax already run to it, but no network cable.
- **Repurposed hardware.** The router is an old desktop, not a dedicated appliance.
- **One physical server.** Everything runs on a single mini PC.
- **Nothing exposed to the internet.** No service should be reachable from outside by default.

---

## 1. My own router instead of the ISP's gateway

The gateway from my ISP can route and hand out addresses, but it gives almost no
visibility or control. I put an **OPNsense** box in front of it and turned the ISP
gateway into an access point — it now only provides Wi-Fi and switch ports, and does
no routing.

I found the interesting part of a home network is deciding 
which device is allowed to talk to which other device, and being able to see what's
happening. A consumer gateway hides all of that.

---

## 2. Separate networks instead of one flat network

Instead of every device sharing one network where anything can reach anything else,
I set up separate interfaces on the router, each with its own address range and its own
firewall rules:

| Interface | What's on it | Address range |
|---|---|---|
| **LAN** | My main network — computers, phone, the server | `192.168.1.0/24` |
| **TAILSCALE** | Devices connecting in from outside the house | — |
| **WAN** | The internet-facing side | from the ISP |

**Why:** on a single flat network, a device that gets compromised has a clear path to
every other device, including the server. Putting the camera base station on its own
interface and its own address range means it isn't on the same network as my computers
and my server. That's the whole point — the default is that devices can't reach each
other, and anything that can is something I allowed on purpose.

**The cost:** inter-device access now has to be deliberate, and when something can't
connect, the reason is usually a rule I wrote — which makes troubleshooting slower than
on a flat network. That's the tradeoff, and it's the one I wanted.

---

## 3. Local DNS with blocklist filtering

Name lookups run on my router using
**Unbound**, with blocklists applied.

**Two reasons.** First, no outside company gets a log of every site my network visits.
Second, filtering happens at the router, so it applies to every device automatically —
no per-device setup, no software installed on phones or laptops.

---

## 4. Using the coax already in the wall

The room I wanted on the network had coax run to it, but no network cable. Rather than
open up a finished wall, I used **MoCA adapters** — ethernet over coax — to get a
network connection there.

**What I got wrong at first:** I assumed the coax was effectively its own separate
connection. It isn't. The adapters join that room to the *same* network as the rest of
the house, just over a different kind of cable.

---

## 5. The VPN router in the far room

I wanted to choose which devices go out through a VPN and which go out directly, rather
than putting the whole house behind one. So I put a **GL.iNet** router in the far room
and set it to route chosen devices through a VPN while the rest go out normally.

### What's actually happening

The GL.iNet's "WAN" port is plugged into my own network, not into the internet. So a device behind it gets its traffic translated
twice — once by the GL.iNet, and again by the main OPNsense router on the way out:

```
far-room PC → GL.iNet → coax adapters → OPNsense → internet
```

It is **two routers in a row on one connection**. I had assumed it was a second,
independent internet feed. Checking the address tables and tracing the route is what
settled it.

### Double-NAT Problem

Devices behind the GL.iNet could reach the internet fine, but **couldn't reach the
services on my server** — unless Tailscale was running on the GL.iNet.

My first guess was to switch the GL.iNet to access-point mode. That would have fixed the
connection, and it would also have prevented choosing
which devices use the VPN. An access point just passes
traffic along — it has no way to send different devices down different paths.

**What was actually happening:** traffic going from devices behind the GL.iNet to my
server isn't leaving the network — it's headed for a private address on the other side
of the GL.iNet's upstream port. Two things worked against it:

1. The GL.iNet had no rule allowing traffic to come back in from its upstream port
   toward its own devices, so that traffic was dropped by default.
2. Its VPN routing rule didn't exclude my own network's address range, so traffic
   meant for my server was being sent into the VPN tunnel instead of straight there.

Tailscale fixed the problem because it added a second path that skipped the GL.iNet's
rules entirely.

---

## 6. Remote access without opening any ports

Remote access uses **Tailscale** rather than port forwarding from my router.

---

## 7. Separating services into their own Docker Compose files

Rather than one giant configuration file for everything, each service has its own
Docker Compose file, and they share a common Docker network. 

**Why separate files:** a service can be updated, restarted, or taken down without
touching anything else. One big file ties every service's fate together.
