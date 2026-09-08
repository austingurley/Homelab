# Domain Controller Setup — DC01

## Why this VM, built this way

I installed Windows Server 2025 with the Desktop Experience (the full GUI) rather than Server Core for this first domain controller. Server Core is lighter — smaller attack surface, fewer patches, less overhead — but everything on it is managed through PowerShell with no local desktop. For a first AD build, I wanted the GUI's discoverability (Server Manager, snap-ins like Active Directory Users and Computers) while I'm still learning where everything lives. Server Core is a reasonable target for a second DC down the line, once the workflow is familiar.

VM specs (2 cores, 4GB RAM, 64GB disk) were set based on Microsoft's published minimum/recommended requirements for a domain controller, balanced against what my host (pve01, 16GB total RAM) could actually spare alongside the other VMs and containers already running.

I enabled the TPM (Trusted Platform Module) even though Windows Server doesn't require it the way Windows 11 client does. A TPM is a small, isolated store for cryptographic keys and secrets — it's what BitLocker uses to keep its encryption key separate from the disk it's protecting, and what Credential Guard uses to isolate credential material from the rest of the OS. Proxmox emulates it in software (`swtpm`) at zero performance cost, so there was no reason not to include it, even if I'm not using BitLocker or Credential Guard yet — it leaves the door open if I want to practice either later in the lab.

One disk decision worth explaining: the CD/DVD drive holding the install media stayed on the IDE bus, while the OS disk went on VirtIO. IDE is understood natively by every version of Windows with zero drivers — exactly what you want for install media, since the installer has to be able to read the disc before any drivers are loaded. VirtIO, by contrast, needs a driver before Windows can see it at all, but in exchange gives meaningfully better disk I/O performance than emulated IDE/SATA. That tradeoff — a driver-free bus for the one-time job of booting install media, versus a faster bus (worth the driver step) for the disk the OS will actually run on every day — is why they ended up on different buses.

## The driver troubleshooting story

See [`troubleshooting/issues-and-fixes.md`](../troubleshooting/issues-and-fixes.md#disk-not-appearing-during-windows-server-2025-installation) for the full diagnostic path on the disk-not-detected issue during install (loading the wrong VirtIO driver — `vioscsi` instead of `viostor` — because the disk was actually attached as `virtio0`, not `scsi0`).

Worth stating plainly here: if I hadn't solved that, the installation would never have started at all — there'd be no OS, and therefore nothing to promote to a domain controller. Everything downstream in this document depends on that fix having worked.

## Network prerequisites (before promotion)

A domain controller needs a static IP because of how AD clients actually find it. When a Windows machine needs to authenticate, apply Group Policy, or query AD, it doesn't have the DC's address memorized — it looks up specific DNS SRV records (things like `_ldap._tcp.corp.home.arpa`) that point to the DC by IP. If that IP changes, every one of those DNS records is now pointing at the wrong address, and authentication, Group Policy, and replication all silently break until DNS is fixed. A workstation getting a new DHCP lease is a non-event; a domain controller getting a new IP is an outage.

On the Proxmox side, `vmbr1` itself was left with no IP address, no gateway, and no bridge ports. That's because a bridge is a Layer 2 device — it forwards Ethernet frames between whatever's plugged into it, the same way a physical switch does, and a switch doesn't need an IP address to do that job. IP addressing is a Layer 3 concept that belongs on the endpoints actually communicating — in this case, the DC's own network adapter — not on the switch connecting them. With no bridge ports configured, `vmbr1` also isn't connected to any physical NIC or to the Proxmox host's own network stack, which is exactly what keeps this network isolated: right now it's a private switch that only VMs plugged into it can use.

I landed on `192.168.50.0/24` for this network, deliberately different from my home LAN's `192.168.1.0/24`, even though this network can't reach the internet yet. The reason is forward-looking: once OPNsense goes in to route between the lab and my home network, having two subnets that don't overlap is what makes that routing possible at all — two networks with the same address range can't be routed between without extra NAT complexity. Within that `/24`, I assigned DC01 the low address `.10`, following the same convention my home network already uses — reserving the low end of the range for static, infrastructure-role addresses and leaving the higher end open for a future DHCP pool, rather than letting static and dynamic assignments collide.

## Choosing the domain name

`corp.home.arpa` breaks into two meaningful parts. `home.arpa` is a domain suffix formally reserved by RFC 8375 specifically for home networks — the IETF guarantees it will never be sold or delegated as a real, publicly resolvable domain, which avoids two problems common in AD lab tutorials: using `.local` (which conflicts with mDNS/Bonjour traffic) or using a real domain you don't own (which risks collision if that domain is ever registered). `corp` is the prefix real enterprises commonly use for their internal AD namespace — naming the internal domain `corp.<something>` rather than reusing the public-facing company domain, to keep internal AD DNS cleanly separated from anything resolvable on the public internet. Using it here mirrors an actual enterprise naming convention rather than an arbitrary lab name.

This is one of the few decisions in the whole build that's genuinely expensive to reverse. Once a forest is created, its name is baked into the security identifiers, DNS namespace, certificate templates, and trust relationships tied to it — changing it later isn't a quick edit, it's closer to rebuilding the forest from scratch. Worth getting right up front, which is why I paused to confirm it before running the promotion wizard.

## The promotion itself

I selected **Add a new forest** in the AD DS Configuration Wizard rather than adding a domain to an existing forest, because there was no existing AD environment to extend — DC01 is the first and only domain controller in this lab, so it has to be the root of a brand-new forest.

During setup, the wizard requires a **DSRM (Directory Services Restore Mode) password** — a credential completely separate from the local Administrator password. DSRM is a special recovery boot mode for a domain controller, used if the AD database itself ever needs offline repair or restoration; it operates outside the normal running AD environment. Keeping it separate from the day-to-day admin password is a deliberate security boundary: a compromised admin account doesn't automatically hand over the ability to tamper with the AD database offline, and this "break glass" credential isn't something used often enough to risk being the same password typed daily.

Setup also threw a DNS delegation warning — that a delegation for this DNS server couldn't be created. That warning exists for domains that are subdomains of a real, publicly delegated parent zone, where the parent's nameserver would normally need an NS record pointing down to the new DNS server. Since `corp.home.arpa` sits under a reserved special-use namespace with no parent DNS server to delegate from, there's nothing to delegate in the first place, and the warning is expected and safe to ignore.

After the reboot, the sign-in screen itself was the first confirmation the promotion worked — it now defaulted to signing in as `CORP\Administrator` instead of the local machine context, meaning the machine's identity had shifted from "a standalone server named DC01" to "a domain controller for the corp.home.arpa domain," before I'd run a single diagnostic command.

## Verifying it actually worked

`dcdiag` runs a battery of specific health tests against the domain controller, and a handful are worth knowing individually rather than just seeing "passed test" scroll by:

- **Advertising** confirms the DC is actively announcing itself via DNS/Netlogon so other machines can discover it exists at all. If this failed, clients simply wouldn't know this DC was available to authenticate against.
- **Netlogons** verifies the Netlogon service is reachable and that permissions on the Netlogon share (used for logon processing and scripts) are correct. A failure here would mean users could be unable to log on even though the DC otherwise looks healthy.
- **SysvolCheck** confirms the SYSVOL share — where Group Policy Objects and logon scripts actually live — is present and shared properly. If this failed, Group Policy simply wouldn't replicate or apply anywhere in the domain.

Running `nslookup corp.home.arpa` separately from `dcdiag` checks a different layer entirely. `dcdiag` validates that the DC's internal AD-related services and roles are healthy; it doesn't independently prove that a client on the network can actually resolve the domain name through DNS. Since nearly everything in AD — locating a DC, finding the nearest Global Catalog, Kerberos ticket requests — depends on DNS SRV record lookups working correctly, a DC that passes every `dcdiag` test but can't be resolved by name is still useless to clients. The successful `nslookup` result confirming `corp.home.arpa` resolves to `192.168.50.10` is the piece of evidence that DNS itself, not just the DC role, is functioning end-to-end.

## What's next

With DC01 verified and healthy, the next steps on the project checklist are designing an OU structure (separating admin accounts from regular users, and organizing by department to mirror a real company), creating the first user and group accounts, and eventually joining a Windows 11 client to the domain to test authentication from the client side.