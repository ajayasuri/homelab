# Homelab

A self-hosted homelab built from owned hardware, one understood step at a time.

The point of this lab is not to have services running. It is to understand
what makes them run. Every piece here was built by hand before reaching for
anything that would automate it away, so the abstractions later are choices
rather than magic. That meant rolling a WireGuard tunnel from raw config before
touching a mesh manager, and editing a package repo by hand instead of pasting
a helper script. Slower on purpose. The learning is the product.

This README is the front door. The technical record lives in
[`BUILD_LOG.md`](./BUILD_LOG.md), the story and the lessons in
[`HIGHLIGHTS.md`](./HIGHLIGHTS.md), and the addressing scheme and a running
glossary in [`Homelab_Reference.md`](./Homelab_Reference.md).

> Private addresses (`192.168.x.x`, `10.10.10.x`) are shown throughout. They
> are RFC1918 ranges every lab uses and teach more than they expose. Public
> WAN and carrier addresses, keys, and anything secret are kept out.

---

## What's running now

- **`pve1`** — a Lenovo ThinkPad W540 turned into a headless Proxmox VE node,
  on ZFS, running with the lid shut. A laptop makes a quiet, low-power server
  with a battery for a built-in UPS.
- **`vpn1`** — an unprivileged Debian LXC running a hand-built WireGuard hub.
  With a port forward and a split-tunnel client, the lab is reachable from a
  phone on cellular, anywhere.

That is the whole foundation: a permanent node, and a way to reach it from
outside the house. Everything past this point is the next layer, not the base.

---

## Architecture at a glance (current, flat v1)

```
                       Internet
                          |
                   [ edge router ]      home WAN: public IPv4, confirmed not CGNAT
                          |             UDP 51820 forwarded inward to .2
                  192.168.1.0/24  (flat, no VLANs yet)
                          |
            +-------------+--------------------+
            |                                  |
     pve1  192.168.1.10                  household devices
     Proxmox VE on ZFS, headless
            |
     vpn1  192.168.1.2   (LXC, CT 102)
     WireGuard hub  wg0   10.10.10.0/24
            ^
            |   encrypted tunnel
            |
     phone / laptop, anywhere
```

The household stays on the untouched consumer router for now, so home internet
is never at risk while the lab grows behind it.

---

## How it got here

Two sessions, both in [`BUILD_LOG.md`](./BUILD_LOG.md):

1. **Node one.** A Windows laptop became a headless Proxmox host in an
   afternoon. BIOS prep for virtualization, a bare-metal install on ZFS, a
   static IPv4 override after the installer quietly tried IPv6, a hand-added
   no-subscription repo, and a first container reachable end to end.
2. **Remote access.** A raw WireGuard hub on its own container, kernel
   WireGuard running inside an unprivileged LXC via a TUN passthrough, a
   split-tunnel client provisioned by QR, and a verified handshake from
   cellular.

---

## What this taught me

- **Foundational before orchestration.** Raw WireGuard teaches the mechanics a
  mesh manager hides: keypairs, the way `AllowedIPs` is both a route and an
  access rule, NAT traversal, the MASQUERADE that lets a client reach the LAN
  as the hub. A portal can come later, and now it will not be a black box.
- **Single-disk ZFS is not the scary striping.** The warnings about RAID0 are
  about losing data across multiple disks. On one disk it is no riskier than
  ext4, and choosing it now means a second drive turns the pool into a mirror
  later with no reinstall. Treating the box as permanent flipped the decision.
- **A container shares the host kernel.** Watching the LXC report the exact
  same kernel as the host made the whole model click, and explained why
  WireGuard needed the host's TUN device passed in rather than its own.
- **IPv6 has a mind of its own.** SLAAC built a global address from a router
  advertisement even with Proxmox set to assign none, because not assigning
  and not accepting are two different switches.
- **Flashing is not copying.** A drag-and-dropped ISO will not boot. Etching
  it properly will.
- **WireGuard is silent by default.** It never answers a packet that is not
  cryptographically valid, so the forwarded port is invisible to a scanner in
  a way an open TCP service never is.

---

## Repo layout

```
.
├── README.md                 you are here
├── BUILD_LOG.md              dated technical record of each build session
├── HIGHLIGHTS.md             the story and the moments worth keeping
├── Homelab_Reference.md      IP scheme + a running glossary of every concept
└── configs/
    ├── wireguard/            wg0 hub + client, placeholder keys
    └── proxmox/              network, headless lid drop-in, LXC TUN passthrough
```

Every committed config is a `*.sample` with placeholders, or carries no secret
at all. Real key-bearing files stay local and never reach this repo.

---

## Where it's going

An evergreen build, grown a layer at a time:

- Bring a managed switch in as the lab backbone.
- Segment the lab onto its own VLANs behind OPNsense doing router-on-a-stick,
  using the same blast-radius and lateral-movement thinking a network gets at
  work. The WireGuard endpoint moves here.
- Ingress and identity: dynamic DNS on an owned domain, a reverse proxy with
  TLS, single sign-on in front of services.
- Real services behind that identity layer: a password manager, media, and
  self-hosted file sync.
- Monitoring across the host and everything on it.

This overlaps almost entirely with a security-engineering skill set, which is a
welcome bonus, but the reason it gets built is that I want to understand how all
of it actually works.

---

*Lab home: `pve1.adytum.dev`. The name comes from "adytum," the innermost
chamber of a temple, the room only the owner enters. It fit a lab you own end
to end.*
