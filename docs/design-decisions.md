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

**Why:** the interesting part of a home network isn't the connection, it's deciding
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
| **ARLO_BASE_STATION** | Only the camera base station | `192.168.10.0/24` |
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

Name lookups (turning a website name into an address) run on my router using
**Unbound**, with blocklists applied.

**Two reasons.** First, no outside company gets a log of every site my network visits.
Second, filtering happens at the router, so it applies to every device automatically —
no per-device setup, no software installed on phones or laptops.

**The cost:** I now maintain the DNS resolver myself. When a blocklist is too
aggressive, sites break in ways that look like a network problem but aren't.

---

## 4. Using the coax already in the wall

The room I wanted on the network had coax run to it, but no network cable. Rather than
open up a finished wall, I used **MoCA adapters** — ethernet over coax — to get a
network connection there.

**What I got wrong at first:** I assumed the coax was effectively its own separate
connection. It isn't. The adapters join that room to the *same* network as the rest of
the house, just over a different kind of cable. That misunderstanding cost me real
troubleshooting time later, described in the next section.

---

## 5. The VPN router in the far room

I wanted to choose which devices go out through a VPN and which go out directly, rather
than putting the whole house behind one. So I put a **GL.iNet** router in the far room
and set it to route chosen devices through a VPN while the rest go out normally.

### What's actually happening

The GL.iNet's "WAN" port is plugged into my own network, not into the internet. Its
address is an address *inside* my LAN. So a device behind it gets its traffic translated
twice — once by the GL.iNet, and again by the main OPNsense router on the way out:

```
far-room PC → GL.iNet → coax adapters → OPNsense → internet
```

It is **two routers in a row on one connection**. I had assumed it was a second,
independent internet feed. Checking the address tables and tracing the route is what
settled it.

### The problem this caused

Devices behind the GL.iNet could reach the internet fine, but **couldn't reach the
services on my server** — unless Tailscale was running on the GL.iNet.

My first guess was to switch the GL.iNet to access-point mode. That would have fixed the
connection, and it would also have **destroyed the thing I built it for**: choosing
which devices use the VPN is something a router does. An access point just passes
traffic along — it has no way to send different devices down different paths. Fixing the
symptom that way would have silently removed the feature.

**What was actually happening:** traffic going from devices behind the GL.iNet to my
server isn't leaving the network — it's headed for a private address on the other side
of the GL.iNet's upstream port. Two things worked against it:

1. The GL.iNet had no rule allowing traffic to come back in from its upstream port
   toward its own devices, so that traffic was dropped by default.
2. Its VPN routing rule didn't exclude my own network's address range, so traffic
   meant for my server was being sent into the VPN tunnel instead of straight there.

Tailscale masked the problem because it added a second path that skipped the GL.iNet's
rules entirely.

**The fix** (not yet applied): exclude my LAN's address range from the VPN routing rule
so local traffic never enters the tunnel, and add one narrow rule allowing the GL.iNet's
upstream port to reach my server's address. Both are small; the hard part was working
out what was wrong.

**Why I haven't applied it yet:** the workaround works, that segment is isolated by
design, and the change touches the routing on the one part of the network I depend on.
I'd rather make it deliberately than in a hurry.

---

## 6. Remote access without opening any ports

Remote access uses **Tailscale** rather than forwarding a port from my router.

**Why:** forwarding a port puts a service on the public internet and makes every
scanner out there my problem. Tailscale means there's no inbound opening at all —
a device has to be added to the network before it can reach anything. It also keeps
working even if my ISP changes my address, which port forwarding does not.

**The cost:** access depends on Tailscale's service being up, and each device needs
the app installed.

---

## 7. Separating services into their own Docker Compose files

Rather than one giant configuration file for everything, each service has its own
Docker Compose file, and they share a common Docker network. Service data is stored
outside the containers so replacing or rebuilding one never risks the data.

**Why separate files:** a service can be updated, restarted, or taken down without
touching anything else. One big file ties every service's fate together.

**Where the passwords live:** not in the Compose files. Each service reads its
credentials from its own `.env` file, which is excluded from this repository. The
templates under [`examples/`](../examples/) show the structure with the values removed.
