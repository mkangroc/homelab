# Design Decisions

This document records **why** the network is built the way it is, what broke along
the way, and what I would change. Configuration lives elsewhere and is deliberately
not published.

---

## Constraints I was designing within

- **No new cabling.** The building's coax was already run; ethernet to the far room was not.
- **Repurposed hardware.** The router is an old Dell Inspiron Desktop with a dual-port NIC, not an appliance.
- **One physical server.** Capacity and power budget live on a single mini PC.
- **No inbound ports.** Nothing should be reachable from the internet by default.

---

## 1. Dedicated router/firewall instead of the ISP gateway

The ISP gateway can route and hand out DHCP, but it offers very little visibility
or policy control. I put an **OPNsense** box in front of it and demoted the gateway
to access-point/bridge mode.

**Why:** the interesting part of a home network isn't connectivity, it's *policy* —
which device may reach which other device, and how I can see what's happening.
Consumer gateways abstract that away. OPNsense exposes it.

**Cost:** one more device to maintain, and if it fails the whole network is down.

---

## 2. Zone segmentation instead of one flat LAN

Devices sit in separate zones with distinct firewall policy rather than sharing
one trust domain:

- **Server zone** — the mini PC and its services.
- **Client zone** — workstations, phones, the camera base station.
- **Egress zone** — the policy-routed segment described in section 5.

**Why:** a flat LAN means every device can reach every other device. That's
convenient and wrong — a compromised device on a flat network has direct access to
the service host. Segmentation means the default is deny, with explicit allows.

**Tradeoff:** more rules to write and maintain, and inter-zone access has to be
intentional. Debugging a blocked connection takes longer when the default is deny —
which is the point.

---

## 3. Local DNS with blocklist filtering (Unbound)

Resolution runs on the router via **Unbound**, with blocklists applied.

**Why:** two reasons. First, no third-party resolver sees a log of every domain
requested on the network. Second, filtering happens at the resolver, so it applies
to every device without per-device configuration or an agent.

**Tradeoff:** I now maintain the resolver. A misconfigured blocklist produces
"sites are broken" symptoms that look like network faults but aren't.

---

## 4. Coax as the layer-2 backhaul to the far room

The far room had coax already run to it. Rather than pull new cable, I used
**MoCA** (ethernet over coax) to extend the LAN there.

**Why:** the marginal cost was one adapter versus a cable run through a finished
wall.

**Important consequence I did not anticipate:** the MoCA segment is not a separate
network. It is *bridged into the same broadcast domain* as the main LAN. I initially
reasoned about the far-room router as though it were on its own uplink; it is
actually layer-2 adjacent to everything on the LAN. That misconception cost me
real debugging time (see section 5).

---

## 5. Policy-based routing on a downstream router

I wanted **per-device egress selection** — choose which devices exit through a VPN
tunnel and which go direct — so I put a **GL.iNet** router in the far room and
configured client-based VPN profiles with policy routing.

This is the most interesting and most instructive part of the build, and it produced
the best debugging story.

### What the topology actually is

The GL.iNet's "WAN" port is fed from the MoCA adapter, and its WAN address is an
address **inside the main LAN subnet**. That means:

```
Fedora → GL.iNet (NAT #1) → MoCA bridge → gateway → OPNsense (NAT #2) → ONT → internet
```

It is **double NAT on a single ingress path** — not, as I first assumed, a second
independent internet feed. I only settled this by inspecting ARP tables and running
traceroutes: the GL.iNet appeared as a layer-2 neighbour on the server's own
interface, with its gateway address pointing at the main router.

### The problem this created

Devices behind the GL.iNet could reach the internet normally, but could **not** reach
services on the mini PC — unless a mesh VPN client was running on the GL.iNet.

At first I assumed the fix was to switch the GL.iNet to access-point mode. That
would have worked, and it would also have **destroyed the entire feature**: policy
routing is a layer-3 router function. An access point bridges frames; it cannot mark
packets or choose between tunnels. The GL.iNet has to be *routing* for per-device
VPN selection to exist at all.

### The actual cause

The behaviour was not a mode problem. Traffic from the GL.iNet's LAN to the mini PC
is forwarded traffic destined for a private subnet upstream of its WAN. Two things
were working against each other:

1. **No allow rule for WAN→LAN-initiated forwarding** on the GL.iNet, so
   LAN-destined traffic arriving on its WAN interface was dropped by default; and
2. **the VPN policy rule had no local-subnet exclusion**, so traffic destined for the
   main LAN subnet was being swept into the tunnel instead of taking the direct path.

The mesh VPN worked around it because it added a second path that bypassed the
GL.iNet's forwarding rules entirely.

### What I'd do differently

Add `192.0.2.0/24` as an explicit exclusion in the VPN policy so LAN-destined traffic
never enters the tunnel, then add a narrow WAN→LAN forwarding allow for the server's
address only. Both are small changes; the hard part was the diagnosis, which required
understanding that "WAN" on this device means "a port on my own LAN."

### Why I left it as-is for now

The workaround functions, the segment is isolated by design, and the fix touches the
routing policy for the one part of the network I depend on for VPN egress. I've
documented it rather than rushing it — this is the next change to make.

---

## 6. TLS on internal services via a private CA

Every service sits behind **Caddy** with TLS, including services never exposed to the
internet, using a locally generated certificate authority.

**Why:** the alternative is plaintext HTTP on the LAN, which means credentials cross
the network in the clear. One service in particular (a password manager) genuinely
requires a secure context — its web vault depends on browser Web Crypto, which is
unavailable over plain HTTP. TLS wasn't optional there.

**Tradeoff:** a private CA means every client has to trust the root certificate, and
I own the rotation.

---

## 7. One Compose stack per service, one shared external network

Services are grouped into **9 Docker Compose stacks** on a single shared, externally
defined Docker network, with persistent volumes and health checks.

**Why per-service stacks:** a service can be rebuilt, backed up, or taken down without
touching its neighbours. One giant compose file couples everything's lifecycle
together.

**Why a shared external network:** containers reach each other by service name without
publishing ports to the host. Only the reverse proxy needs to be reachable, and
service ports stay off the host interface.

**Note on secrets:** credentials are supplied from `.env` files via variable
substitution, never inlined in the compose files. Those files are excluded from this
repository; the sanitized templates under `examples/` show the structure with all
values externalized.

---

## 8. Mesh VPN for remote access instead of port forwarding

Remote access uses **Tailscale** rather than exposing services through port forwards.

**Why:** forwarding a port puts a service on the public internet and makes it
everyone's problem. A mesh VPN means there is no inbound listener at all — access
requires the device to be a member of the network. It also survives being behind
carrier NAT, which port forwarding does not.

**Tradeoff:** access depends on the mesh provider's coordination service, and clients
need the agent installed.

---

## 9. Known limitations

These are real and I'd rather state them than let someone assume otherwise:

- **Single point of failure.** One server host. If it goes down, most services go down.
- **No backup pipeline yet.** This is the biggest gap and the next project: a
  snapshot/rotation/verification pipeline for the service data, with a documented and
  actually-tested restore procedure. A backup nobody has restored from isn't a backup.
- **No redundant DNS.** If the resolver is down, name resolution fails network-wide.
- **The section 5 routing fix is outstanding.**
- **Monitoring exists; alerting is minimal.** Uptime Kuma reports availability, but
  nothing pages me.

---

## 10. What this project taught me

The technical material was the easy part. The durable lessons were:

- **Verify topology, don't infer it.** My wrong mental model of the far-room segment
  came from reasoning about what a "WAN" port *should* mean rather than checking what
  it actually was. ARP tables and traceroutes settled in minutes what assumptions had
  muddled for hours.
- **Know which layer a feature lives at before changing modes.** Switching to access
  point mode would have "fixed" connectivity by silently deleting the feature I built
  the device for. A fix that removes the reason the component exists isn't a fix.
- **The default-deny design is working when it's inconvenient.** Blocked traffic that
  I have to reason about is the design doing its job.
