# Active Directory Homelab

I'm turning an old desktop into a virtualized Active Directory and infrastructure lab. The goal is tp build real experience in help desk work, system administration, network administration, and infrastructure engineering — the skills that I need for a Network+ certification and a networking internship.

I'm documenting each phase — what I built, what broke, and how I fixed it — the way you'd document work in a real IT environment.

## What's running right now

The host is a repurposed i7-8700 desktop running Proxmox VE 9.2. It sits on my home network with a static management IP, fully updated and verified. I can reach it remotely through Tailscale, without exposing any ports to the internet.

Full hardware and software details live in `inventory/hardware-software.md`.

## Where this is headed

The plan is a small isolated network behind the Proxmox host: a Windows Server domain controller, one or two Windows 11 clients, and a Linux administration server. Later, an OPNsense firewall will sit between the lab and the internet, and a Tailscale-connected container will let me manage all of it remotely, from anywhere.

You can see the full target architecture in `diagrams/network-topology.png`.

## Status

- [x] Migrate Windows off the drive I'm reusing for Proxmox, and verify it boots independently
- [x] Build the Proxmox host and get it fully updated
- [x] Set up remote access through Tailscale
- [ ] Build the isolated lab network
- [ ] Stand up the domain controller (AD DS, DNS)
- [ ] Join Windows clients to the domain
- [ ] Add a Linux administration server
- [ ] Add OPNsense for routing and firewall rules
- [ ] Test backups and restores

## How this is organized

- `inventory/` — hardware and software details
- `proxmox/` — the host build process
- `networking/` — addressing, DNS/DHCP, firewall rules, remote access
- `active-directory/` — domain controller setup, users, groups, group policy
- `troubleshooting/` — problems I hit and how I solved them
- `screenshots/` — visual proof of each step

If you want the interesting part, skip to `troubleshooting/issues-and-fixes.md`. That's where the real work happened — a subnet mismatch that took actual diagnostic steps to track down, not just a checklist I followed in order.
