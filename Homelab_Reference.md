# Homelab Reference

Running reference for the homelab build. Two sections: the IP addressing
scheme (so addresses are never random) and a key-terms log for
self-directed research. Append as the build grows. Drops cleanly into the
Git repo or the vault later.

---

## IP addressing scheme

Two layers. The host-numbering bands get adopted now. The VLAN and subnet
map is a v2 leaning that firms up at segmentation.

### Host numbering (fourth octet, role bands)
Adopt now. The address tells you what a device is.

- `.1` = gateway (the subnet's router)
- `.2` to `.9` = network infrastructure (VPN edge, firewall mgmt, switch mgmt, APs)
- `.10` to `.19` = hypervisor nodes
- `.20` to `.49` = service VMs and containers
- `.50` to `.254` = DHCP pool (dynamic clients)

### VLAN and subnet map (third octet = VLAN ID)
Pencil for v2. The subnet tells you the segment. Household stays on the
Verizon default, untouched.

- VLAN 1 / `192.168.1.0/24` = household (CR1000B, untouched)
- VLAN 10 / `192.168.10.0/24` = management
- VLAN 20 / `192.168.20.0/24` = servers and services
- VLAN 30 / `192.168.30.0/24` = lab and experimental
- VLAN 40 / `192.168.40.0/24` = IoT and cameras
- VLAN 99 / `192.168.99.0/24` = guest and untrusted

### Current assignments (v1, flat on 192.168.1.0/24)
- `192.168.1.1` = CR1000B (gateway)
- `192.168.1.10` = pve1 (Proxmox host)
- `192.168.1.2` = WireGuard host (proposed, network-infra band)
- `192.168.1.50` to `.254` = DHCP pool

At v2 the lab moves onto its own VLAN and subnet behind OPNsense. The
household keeps `192.168.1.0/24`. Service and node hosts renumber into the
VLAN scheme; the host-numbering bands carry over unchanged.

---

## Key terms and concepts (research log)

Term, one-line gist, and what to dig into on your own.

- **WireGuard:** modern in-kernel VPN protocol. Research: how it compares to IPsec and OpenVPN, why it is smaller and faster.
- **Cryptokey routing:** WireGuard's model where a peer's public key maps to its allowed IPs. Research: how AllowedIPs is both routing and ACL.
- **AllowedIPs:** the field deciding what a peer may send and what gets routed to it. Research: the double role, and the `0.0.0.0/0` full-tunnel case.
- **Hub-and-spoke vs mesh:** home as the central hub vs every node talking directly. Research: when a mesh (Tailscale, Headscale, Netbird) earns its place.
- **CGNAT (carrier-grade NAT):** ISP sharing one public IP across many customers. Research: the `100.64.0.0/10` range, why it blocks inbound, how to detect it.
- **Port forwarding:** a router rule sending an inbound port to an internal host. Research: NAT, why WireGuard uses UDP.
- **WireGuard silent-by-default:** no reply to packets that are not cryptographically valid. Research: how this looks to a port scanner vs an open TCP service.
- **OPNsense:** open-source firewall and router OS (FreeBSD). Research: vs pfSense, what a software firewall does that a consumer router cannot.
- **Router-on-a-stick:** one trunk link carrying multiple VLANs into a router that routes between them. Research: subinterfaces, inter-VLAN routing.
- **VLAN / 802.1Q:** logical network segmentation using tagged frames. Research: tagging, the VLAN ID, broadcast domains.
- **Trunk vs access port:** trunk carries many tagged VLANs, access carries one untagged. Research: the native VLAN, where each is used.
- **PoE / PoE+:** power delivered over the ethernet cable. Research: wattage classes, what PoE+ can power.
- **LXC vs VM:** a container shares the host kernel, a VM runs its own. Research: when to use each, why Docker prefers a VM here.
- **TUN/TAP device:** a virtual network interface the kernel exposes for VPNs. Research: TUN (layer 3) vs TAP (layer 2), why WireGuard needs TUN.
- **Privileged vs unprivileged LXC:** whether the container's root maps to host root. Research: user namespace remap, the security tradeoff, device passthrough.
- **ZFS, zpool, mirror, zpool attach:** the filesystem and pool model on pve1. Research: RAID0 vs mirror (RAID1) vs RAIDZ, attaching a disk in place.
- **DDNS (dynamic DNS):** keeps a domain pointed at a changing home IP. Research: how an updater talks to the registrar API.
- **Reverse proxy / TLS termination:** one entry point routing to internal services and handling certificates. Research: Zoraxy, Let's Encrypt, SNI.
- **Authentik / Keycloak:** self-hosted single sign-on and identity providers. Research: SSO, OIDC, SAML, why Authentik is lighter and Keycloak more enterprise.
- **SLAAC:** stateless address autoconfiguration, how a host builds its own IPv6 address from a router advertisement. Research: why it ran even with Proxmox IPv6 set to none.
- **Router Advertisement (RA):** the IPv6 message a router broadcasts that triggers SLAAC. Research: `accept_ra`, the difference between not assigning IPv6 and not accepting RAs.
- **IP forwarding (`net.ipv4.ip_forward`):** the switch that lets a host route packets between interfaces. Research: why a VPN hub needs it on.
- **NAT / MASQUERADE:** rewriting source addresses so tunnel clients reach the LAN as the hub. Research: SNAT vs MASQUERADE, the road-warrior pattern, why no LAN routes are needed.
- **iptables vs nftables:** the older and newer Linux firewall front ends, with `iptables-nft` being the compatibility bridge. Research: which Debian uses by default, how wg-quick picks one.
- **wg-quick (PostUp / PostDown):** the wrapper that brings a WireGuard interface up from a config and runs hook commands. Research: what it does that raw `wg` does not.
- **wg syncconf:** applies config changes to a live WireGuard interface without dropping it. Research: vs `wg setconf` and a full down/up.
- **PersistentKeepalive:** periodic packets that hold a NAT mapping open from behind a client's network. Research: why a road-warrior client behind NAT needs it.
- **Split tunnel vs full tunnel:** client AllowedIPs covering only the home subnets vs `0.0.0.0/0` for all traffic. Research: the privacy and routing tradeoff on untrusted wifi.
- **Hairpin NAT / NAT loopback:** reaching your own public IP from inside the LAN. Research: why many consumer routers fail it, why you test a VPN from cellular.
- **Namespaced vs non-namespaced sysctls:** why some sysctl keys set fine in an unprivileged container and others return permission denied. Research: network namespace ownership.
- **Kernel module / modprobe:** loading kernel features like `tun` on the host. Research: why a container uses the host's kernel and modules, never its own.
- **qrencode:** renders a text config as a scannable QR for mobile import. Research: how the WireGuard mobile app consumes it.
