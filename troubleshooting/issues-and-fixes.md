## Issue 1: Proxmox Unreachable After Relocating the Host

**Symptom:** After moving the Proxmox host to a new location and reconnecting it to power and Ethernet, I couldn't reach the web UI or ping it at its known IP address, even though the Ethernet link lights on both ends showed a live connection.

**Diagnostic steps:**
1. Confirmed the host was powered on and the Ethernet link lights were lit, ruling out a dead cable or power issue.
2. Pinged the known IP (`192.168.100.2`) — no response.
3. Checked my router's admin page for a device list — nothing matching showed up at all.
4. Checked my own Mac's actual IP and gateway, and discovered it was on a completely different subnet (`192.168.1.x`) than the address I was trying to reach.
5. Found the real router admin page at the correct gateway and confirmed the actual home network was `192.168.1.0/24` — a different network entirely from `192.168.100.x`.

**Root cause:** Proxmox's network configuration is static, not DHCP — it was still configured for the network it was originally installed on, which was a different subnet than the one it was now physically connected to after the move. Since it never requests a new address automatically, it kept trying to use an address that simply didn't exist on its new network.

**Fix:** Logged into the local console directly (the only way in, since the web UI was unreachable), edited `/etc/network/interfaces` to update the static IP and gateway to match the actual home network, updated `/etc/resolv.conf` for DNS, and rebooted.

**What I'd check first next time:** If a device becomes unreachable right after being physically moved, check whether it's using a static IP configuration before assuming it's a cable or hardware problem — a static config doesn't care that the physical network around it changed. Knowing my home network's actual subnet and gateway ahead of time would have made this diagnosis faster too.


## Issue 2: Missing `curl` on the Debian LXC Template

**Symptom:** After apparently running Tailscale's install script inside the `tailscale01` container, `tailscale up` returned "command not found" — as if nothing had installed at all.

**Diagnostic steps:**
1. Checked whether Tailscale had actually installed with `which tailscale` and `dpkg -l | grep tailscale` — both came back empty.
2. Re-ran the install process, but this time split it into two steps instead of piping straight to `sh`, specifically to see any error output that direct piping might hide.
3. That revealed the real error: `bash: curl: command not found`.

**Root cause:** The minimal Debian 12 LXC template doesn't include `curl` by default. Tailscale's official install script starts with a `curl` command to fetch itself, so it failed at the very first step — and because the original command piped `curl` directly into `sh`, that failure was silent instead of producing a visible error.

**Fix:** Installed `curl` with `apt install -y curl`, then re-ran Tailscale's install script, which completed successfully.

**What I'd check first next time:** When a command fails with "command not found" right after supposedly installing something, verify the install process actually completed rather than assuming it did — piping a command directly into `sh` can hide a failure that would otherwise be obvious. It's also worth running a quick `apt update && apt install -y curl` right after spinning up any minimal container template, before assuming standard tools are already there.


## Issue 3: LXC Template Download Failing Over IPv6

**Symptom:** Downloading the Debian 12 LXC template on Proxmox failed with "TASK ERROR: download failed... Network is unreachable," while trying to connect to an IPv6 address.

**Diagnostic steps:** Read the exact task log, which showed Proxmox resolving `download.proxmox.com` to an IPv6 address (`2a0b:7140:8:100::90`) and failing to connect over it.

**Root cause:** The Proxmox host had auto-configured itself an IPv6 address, but my home network doesn't actually provide a working IPv6 route to the internet. Since `download.proxmox.com` has both an IPv4 and IPv6 address, and Linux tries IPv6 first by default when both are available, the download attempt tried the non-functional IPv6 path and failed — even though IPv4 would have worked the whole time.

**Fix:** Disabled IPv6 entirely on the Proxmox host, since nothing in this build uses it, by adding a permanent sysctl setting and applying it immediately. Retried the template download afterward, and it succeeded over IPv4.

**What I'd check first next time:** When a download or connection fails with "Network is unreachable" to an address that looks unusual, check whether it's actually trying IPv6 instead of IPv4 — a common issue on networks without full IPv6 support, and worth ruling out early rather than assuming a broader connectivity problem.


## Issue 4: Corrupted Tailscale Login Link

**Symptom:** Opening the authentication link printed by `tailscale up` resulted in `DNS_PROBE_FINISHED_NXDOMAIN` — the browser couldn't resolve the domain at all, not even a failed connection or an error page from Tailscale itself.

**Diagnostic steps:**
1. Tried the same link on a different device (phone) — same failure, ruling out a problem specific to one machine.
2. Loaded the plain `tailscale.com` homepage on the same devices — that worked fine, isolating the problem to this specific link rather than a broader DNS issue.
3. Checked Tailscale's official status page to rule out an outage on their end — everything showed operational.

**Root cause:** The link had gotten corrupted when I copied it out of the browser-based console — likely a hidden character introduced in the copy — which made the browser try to resolve a hostname that didn't actually exist, producing NXDOMAIN rather than a normal connection failure.

**Fix:** Ran `tailscale up` again to generate a fresh link, and this time typed the address manually instead of copying and pasting it, which avoided whatever was corrupting the copy.

**What I'd check first next time:** If a copied link produces an outright "domain doesn't exist" error rather than a normal failed connection, suspect the copy itself before assuming the service is down — regenerating and manually typing the link is a fast, harmless way to rule that out.