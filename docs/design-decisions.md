# Design Decisions

This document records **why** the network is built the way it is, what broke along
the way, and what I would change.

---

## Constraints I was designing within

- **No new cabling.** The building's coax was already run; ethernet to the far room was not.
- **Repurposed hardware.** The router is an old desktop with a dual-port NIC, not an appliance.
- **One physical server.** Capacity and power budget live on a single mini PC.
- **No inbound ports.** Nothing should be reachable from the internet by default.

---

## 1. Dedicated router/firewall instead of the ISP gateway

The ISP gateway can route and hand out DHCP, but it offers very little visibility
or policy control. I put an **OPNsense** box in front of it and demoted the gateway
to access-point/bridge mode.

---

## 2. Zone segmentation instead of one flat LAN

Devices sit in separate zones with distinct firewall policy rather than sharing
one trust domain:

- **Server zone** — the mini PC and its services.
- **Client zone** — workstations, phones, tablets.
- **Egress zone** — the policy-routed segment described in section 5.

---

## 3. Local DNS with blocklist filtering (Unbound)

Resolution runs on the router via **Unbound**, with blocklists applied.

Two reasons. First, no third-party resolver sees a log of every domain
requested on the network. Second, filtering happens at the resolver, so it applies
to every device without per-device configuration or an agent.

---

## 4. Coax as the layer-2 backhaul to the far room

The far room had coax already run to it. Rather than pull new cable, I used
**MoCA** (ethernet over coax) to extend the LAN there.

**Important consequence I did not anticipate:** the MoCA segment is not a separate
network. It is *bridged into the same broadcast domain* as the main LAN. I initially
reasoned about the far-room router as though it were on its own uplink; it is
actually layer-2 adjacent to everything on the LAN. 

---

## 5. Policy-based routing on a downstream router

I wanted **per-device egress selection** — choose which devices exit through a VPN
tunnel and which go direct — so I put a **GL.iNet** router in the far room and
configured client-based VPN profiles with policy routing.

### What the topology actually is

The GL.iNet's "WAN" port is fed from the MoCA adapter, and its WAN address is an
address **inside the main LAN subnet**. That means:

```
Workstation PC → GL.iNet (NAT #1) → MoCA bridge → gateway → OPNsense (NAT #2) → ONT → internet
```

It is **double NAT on a single ingress path**. 

### The problem this created

Devices behind the GL.iNet could reach the internet normally, but could **not** reach
services on the mini PC — unless a mesh VPN client (Tailscale) was running on the GL.iNet.

At first I assumed the fix was to switch the GL.iNet to access-point mode. That
would have worked, and it would also have **terminated the entire feature**.

---

## 6. TLS on internal services via a private CA

Every service sits behind **Caddy** with TLS, including services never exposed to the
internet, using a locally generated certificate authority.

The alternative is plaintext HTTP on the LAN, which means credentials cross
the network in the clear. One service in particular (Vaultwarden) genuinely
requires a secure context — its web vault depends on browser Web Crypto, which is
unavailable over plain HTTP. TLS wasn't optional there.

---

## 7. One Compose stack per service, one shared external network

Services are grouped into **9 Docker Compose stacks** on a single shared, externally
defined Docker network, with persistent volumes and health checks.

I chose a per-service approach because they can be rebuilt, backed up, or taken down without
touching its neighbours. One giant compose file couples everything's lifecycle
together.

---

## 8. Mesh VPN for remote access instead of port forwarding

Remote access uses **Tailscale** rather than exposing services through port forwards.
