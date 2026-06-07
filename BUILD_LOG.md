# Homelab Build Log

A dated technical record of building a self-hosted homelab from owned hardware:
a Proxmox node, a hand-rolled WireGuard VPN for remote access, and the network
foundation underneath both. The companion `HIGHLIGHTS.md` carries the story and
the lessons. This file is the what-was-built record.

Private addresses (`192.168.x.x`, `10.10.10.x`) are shown on purpose. They are
RFC1918 ranges every lab uses and reveal nothing useful to an attacker. Public
WAN and carrier addresses are redacted throughout.

---

## 2026-06-05 — Planning and hardware roster

Decided the shape of the build before touching hardware.

- **OS path: Proxmox VE, bare metal.** No separate Linux install step. Proxmox VE
  is Debian plus the KVM and LXC stack. Docker, when it arrives, runs inside a
  Linux VM rather than an LXC, which avoids the permission gotchas of Docker in a
  container.
- **Node one: Lenovo ThinkPad W540.** A laptop is a server with a built-in UPS,
  quiet and low-power, and ThinkPads have well-trodden Proxmox install paths.
- **Switch: Cisco Catalyst 2960-S PoE+.** Managed, full VLAN support, quiet. PoE+
  can power cameras, APs, and VoIP phones straight off the ports.
- **Edge router stays the consumer Verizon unit for v1**, so household internet is
  never at risk while the lab is built behind it. The ONT-to-router link is
  ethernet, no coax.
- **Firewall path: OPNsense as a VM**, not an old hardware appliance. A modern,
  career-relevant software firewall doing inter-VLAN routing later.
- **Domain: adytum.dev**, registered on Porkbun. Parked until the ingress phase,
  then pointed at the home IP via dynamic DNS.
- **VPN direction: raw WireGuard first**, learn the primitives by hand, add a mesh
  orchestrator only later if the peer count ever justifies it.

---

## 2026-06-06 (morning) — Node one: Proxmox on the W540

A Windows laptop became a headless Proxmox host in roughly six hours, end to end.

**Hardware**
- i7-4800MQ (Haswell, 4 cores / 8 threads, VT-x and VT-d).
- 16GB DDR3 across all four SODIMM slots.
- 224GB SSD, single disk.
- Dual GPU (Intel HD 4600 + NVIDIA Quadro K1100M). No clean discrete-disable in
  this firmware, which is normal for the W540. The internal LCD is the boot
  display, so the installer rendered fine.

**BIOS prep**
- Enabled Intel Virtualization Technology and VT-d (both were off).
- Disabled Secure Boot (the most common reason a flashed USB will not boot on
  this generation).
- Wake on LAN on for AC and battery; network boot device set to the SSD.
- Pinned network interface names for stable NIC naming.

**Install: Proxmox VE 9.2.2, bare metal**
- Filesystem: ZFS, single-disk RAID0 pool (the only ZFS option on one disk).
- The USB had to be reflashed with balenaEtcher. A drag-and-dropped ISO is not
  bootable.
- The installer auto-filled IPv6 from the ISP. Every network field was overridden
  to static IPv4 by hand.

**Network**
- Static `192.168.1.10/24`, gateway `192.168.1.1`, DNS `1.1.1.1`.
- Hostname `pve1.adytum.dev`.

**Update**
- Disabled the enterprise and ceph-enterprise repos, added the no-subscription
  repo by hand rather than running a community helper script blind.
- Full upgrade, rebooted onto kernel `7.0.6-2-pve`.

**Router**
- DHCP range shrunk to `192.168.1.50` through `.254`, carving `.2` to `.49` into a
  static block for cluster infrastructure. No per-device reservations needed.
- Confirmed the static block was empty before the change.

**Headless operation**
- A drop-in `/etc/systemd/logind.conf.d/99-lid.conf` sets the lid-switch handlers
  to ignore. Verified: lid closed, host stays up and reachable.

**v1 done condition met**
- First container: CT 100, hostname `test1`, Debian 13 LXC on DHCP, reachable on
  the LAN and able to reach the internet. Foundation proven end to end.

**Decisions locked**
- **ZFS is the cluster storage standard.** Proxmox replication only runs on ZFS;
  ext4 with LVM cannot replicate. Single-disk RAID0 today is no riskier than ext4
  on one disk and upgrades to a mirror by adding a second drive and running
  `zpool attach`, with no reinstall.
- **The W540 is the permanent first node and anchor, not a throwaway.** The
  cluster grows around it.
- **Static IP scheme:** `.2` to `.49` reserved static, hypervisor nodes in the
  `.10` to `.19` band.
- **Proxmox clusters are peer-to-peer**, not a main host with subordinates.
- **Foundational before orchestration:** the repo fix was done by hand to learn
  it, not via a helper script. Read-first is the habit for any curl-bash script.

---

## 2026-06-06 (evening) — Remote access: raw WireGuard by hand

Built a WireGuard hub-and-spoke VPN from first principles on a dedicated
container, then proved a connection from a phone on cellular reaching the home
network. CGNAT was checked first and cleared: the home WAN address is a real
public IPv4, so plain WireGuard with the house as the hub works and no mesh
overlay is needed.

**Host: `vpn1`, CT 102**
- Unprivileged Debian 13 LXC. CTID 102 chosen to echo the `.2` address.
- Static `192.168.1.2/24`, gateway `.1`, DNS `1.1.1.1`. IPv6 disabled inside the
  container after SLAAC auto-configured a global address despite the GUI set to
  none.
- TUN passthrough via `lxc.cgroup2.devices.allow: c 10:200 rwm` plus a bind mount
  of `/dev/net/tun`. Applied only after a full stop and start, never a soft
  reboot.
- `wireguard-tools` and `iptables` installed with `--no-install-recommends` after
  the package pulled in an unused realtime kernel image.

**WireGuard hub: `wg0`**
- Tunnel subnet `10.10.10.0/24`, hub at `10.10.10.1`, ListenPort `51820/udp`.
- `net.ipv4.ip_forward = 1`, plus a PostUp MASQUERADE rule so tunnel clients reach
  the whole LAN as the hub (the road-warrior pattern, no LAN routes needed).
- `wg-quick@wg0` enabled, persists across reboot.

**First client peer**
- Tunnel IP `10.10.10.2`. Split-tunnel config: AllowedIPs covers `192.168.1.0/24`
  plus `10.10.10.0/24`, so only home-bound traffic tunnels.
- PersistentKeepalive 25 to hold the NAT mapping open from behind the client's
  network. Provisioned to the phone via a terminal QR code.

**Port forward**
- `UDP 51820` to `192.168.1.2`, source any. Safe given WireGuard's silent-by-
  default behavior: it never replies to packets that are not cryptographically
  valid, so the port is invisible to a scanner.

**Verified**
- `wg show` showed the peer connected from a cellular endpoint with a recent
  handshake and transfer in both directions. Tunnel confirmed end to end.

**Decisions locked**
- **IP addressing scheme adopted.** Host-numbering bands now, VLAN-ID-in-third-
  octet subnet map penciled for the segmentation phase.
- **CTID convention:** 100 plus the host's last octet.
- **Tunnel subnet `10.10.10.0/24`**, kept visually distinct from the LAN.
- **Raw WireGuard by hand over a management portal.** A reference setup using a
  WireGuard portal plus identity provider was logged for the later management and
  identity phase, not adopted now. Consistent with foundational-before-
  orchestration.
- **Split tunnel as the client default.** Full tunnel is a one-line change for
  untrusted wifi.
- **Unprivileged LXC plus TUN passthrough** is the chosen WireGuard host pattern.
  Kernel WireGuard works inside it.
- **Lean-container habit:** `--no-install-recommends` going forward.

---

## What's next

- Factory reset and integrate the managed switch as the lab backbone.
- Segmentation: OPNsense as a VM doing router-on-a-stick, VLANs, inter-VLAN
  firewalling. The WireGuard endpoint migrates here.
- Ingress and identity: dynamic DNS for the domain, a reverse proxy with TLS, and
  single sign-on in front of services.
- First real services behind identity: a password manager, media, and self-hosted
  file sync.
- Observability across the host and services.
