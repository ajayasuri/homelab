# Homelab Highlights Logbook

A running log of moments worth keeping, so the blog and the journey story
already have their raw pieces when the writing starts. Story and content
feedstock, not operational notes. Add a dated block each session. Tag what
becomes a post.

---

## 2026-06-06 (morning) — Node one, Windows laptop to Proxmox server

- A used Windows ThinkPad W540 became a headless Proxmox server in six hours, first hardware day.
- The drag-and-dropped USB that would not boot, and learning what flashing an ISO actually means.
- The RAID0 scare: reading that RAID0 is dangerous, then learning RAID needs multiple disks and single-disk RAID0 is not the same animal. A mirror needs a real second drive.
- The ext4-versus-ZFS decision, and how reframing the W540 as permanent flipped the call to ZFS.
- The installer quietly filling in IPv6 from Verizon, caught before committing.
- Watching LXC kernel-sharing prove itself: the container reporting the exact same kernel as the host.
- Debian versus Ubuntu, and why the host's own lineage made Debian the clean container choice.

## 2026-06-06 (evening) — Remote access, raw WireGuard by hand

- IPv6 sneaking back in via SLAAC despite the interface set to none, and learning that Proxmox's "none" only stops Proxmox from assigning, not the kernel from accepting router advertisements.
- The "/dev/net/tun: No such file" puzzle, and the real lesson underneath it: `lxc.mount.entry` is only applied on a full stop and start, never on a soft reboot.
- `wireguard-tools`, an 85KB package, dragging in a whole realtime kernel image the container can never boot. The dead-weight lesson, and the `--no-install-recommends` habit that prevents it.
- Kernel WireGuard creating wg0 inside an unprivileged container, the payoff that made the whole TUN-passthrough detour worth it.
- The finish: a phone out on cellular, completing a handshake with the home hub. Home lab in the pocket, over a tunnel built entirely by hand, no portal, no orchestrator.
- The throughline worth writing: foundational before orchestration. Raw WireGuard first so the abstractions later are understood, not magic. A colleague's wg-portal-plus-Authentik repo sitting in reserve as the proof of where this goes next.
