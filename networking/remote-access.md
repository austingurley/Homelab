## Why Tailscale Instead of Port-Forwarding

Port-forwarding opens a specific port on your home router so it's reachable from anywhere on the internet — not just by you. That creates a direct attack surface: automated bots constantly scan the internet for open ports like Proxmox's 8006, SSH's 22, and RDP's 3389, and they don't need to be targeting you specifically to find it. Any vulnerability in the exposed service, or a weak or reused password, becomes directly reachable by anyone on the internet, all the time.

Tailscale avoids this entirely by not exposing anything to the public internet in the first place. It builds a private, encrypted tunnel only between devices I've explicitly authorized on my own tailnet. Even someone who knows my home's public IP address has no open door to try — there's nothing listening for outside connections at all.


## Subnet Router and Container Isolation

Normally, Tailscale only connects devices that have it installed directly, meaning each gets its own private Tailscale address, and only those devices can reach each other. A subnet router extends that: one Tailscale node advertises an entire IP range it can reach on its local network, so every other device on the tailnet can then reach anything in that range through it — even devices, like my Proxmox host, that will never run Tailscale themselves.

I installed Tailscale inside an LXC container rather than directly on the Proxmox host for two reasons. First, isolation: a VPN tunnel is constantly talking to the outside internet, and I'd rather any bug or vulnerability in it be contained to a small, disposable container than have it running with full access on the same machine controlling every VM and all my storage. Second, practicality: if the container ever breaks or gets misconfigured, I can remove it and build a new one without touching Proxmox itself at all.


## Setup Steps

1. Downloaded a Debian 12 LXC template on Proxmox. This initially failed with a network error because the download tried to use IPv6, which isn't available on my network — I fixed this by disabling IPv6 on the Proxmox host directly (full story in `troubleshooting/issues-and-fixes.md`).
2. Created the container (CT ID 100), named `tailscale01`: Debian 12 template, 6GB disk, 1 core, 512MB RAM, on the same network bridge as Proxmox itself (`vmbr0`).
3. Containers are unprivileged by default, which means Tailscale can't access the TUN network device it needs to create its tunnel. I enabled the nesting feature (`pct set 100 --features nesting=1`) and edited the container's config file directly to explicitly allow that one device (`lxc.cgroup2.devices.allow` and `lxc.mount.entry` for `/dev/net/tun`), then restarted the container for the change to take effect.
4. Inside the container, ran `apt update && apt upgrade`, then tried Tailscale's official install script. It failed immediately because `curl` wasn't installed on this minimal template. Installed `curl`, then reran the script successfully.
5. Enabled IP forwarding (`net.ipv4.ip_forward = 1`) so the container could actually relay traffic on behalf of other devices, not just handle its own.
6. Ran `tailscale up --advertise-routes=192.168.1.0/24 --accept-dns=false`, which printed a login link.
7. Authenticated through that link in a browser, then approved the advertised `192.168.1.0/24` route for `tailscale01` in the Tailscale admin console — routes aren't trusted automatically, so this manual approval is a deliberate security checkpoint, not an extra step you can skip.


## Testing

I'd already confirmed I could reach Proxmox from my Mac over home Wi-Fi, so that wasn't in question. To properly test remote access, I ran two checks under the same network condition rather than trusting a single result.

First, I connected my Mac to my iPhone's hotspot — genuinely off my home network — with Tailscale turned off, and tried to reach Proxmox. That failed, which was the correct and expected result: it confirmed that simply being off the home network doesn't grant access on its own, and ruled out any chance the hotspot was somehow bridging back to my home network some other way.

Then, still on the same hotspot, I turned Tailscale on and tried again. It connected without issue. Testing both the failure and the success under the exact same network condition — only toggling Tailscale — proved specifically that Tailscale was responsible for the access, not some other factor.