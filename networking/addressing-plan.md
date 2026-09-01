## Home Network

- **Gateway / Router:** AT&T residential gateway at `192.168.1.254`
- **LAN Subnet:** `192.168.1.0/24`
- **DHCP Pool Range:** `192.168.1.64`–`192.168.1.253` (dynamically assigned to clients like phones and laptops)

## Proxmox Static IP

I gave the Proxmox host a static IP of `192.168.1.50`, chosen specifically because it falls below `.64`, which is the start of the router's DHCP pool. That guarantees the router will never hand this exact address out to another device automatically, without needing a separate DHCP reservation to protect it.

## Why a Server Needs a Static IP

A DHCP lease usually returns the same address to a device, but nothing guarantees it stays fixed forever. That's the actual issue for a server: other things depend on finding it at one specific, predictable address. I have a bookmark going straight to `https://192.168.1.50:8006`. My Tailscale subnet router depends on that exact address to route traffic correctly. DNS records and other VMs will point to this same IP later too. If it ever changed, all of that breaks at once, probably at an inconvenient time.

A client like my phone or laptop doesn't have this problem, and nothing else needs to find it at a fixed address, so whatever DHCP hands it works fine for its own outgoing connections.