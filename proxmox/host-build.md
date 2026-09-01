## Why This Hardware

I repurposed my existing PC instead of buying dedicated hardware like a Dell OptiPlex. I already had a working i7-8700 build, and PC component prices are unusually high right now. Buying new hardware for a project I hadn't tested yet didn't make sense when I had a machine sitting idle.

The tradeoff is real, though. This motherboard caps out at 16GB of RAM and doesn't support DDR5, so I gave up an easy upgrade path in exchange for zero cost and hardware I already knew worked. RAM is the actual bottleneck this creates: I can't run every VM in the lab at once without hitting a ceiling, so I'm deliberately limiting how many guests run simultaneously until I upgrade to 32GB.


## Migrating Windows Off the Reused Drive

The homelab PC didn't have a drive in it at all, so storage was the one component missing. I had two SSDs to work with in my main PC: a 1TB drive running Windows, and a 2TB drive holding games and personal files. I don't use much storage day-to-day, so freeing up the 1TB and repurposing it for Proxmox made sense.

Before touching that drive, I cloned the Windows install onto the 2TB SSD and needed to prove the clone actually worked on its own, not just that the cloning tool reported success. I did a full cold shutdown (not a restart, since Windows Fast Startup can mask a boot that isn't really working), physically pulled the 1TB M.2 stick out of the board, and powered back on. Windows booted normally, the drive that used to show as a secondary partition was now correctly `C:`, and Disk Management confirmed the System, Boot, and Active flags had moved to the 2TB drive with everything allocated correctly and no corruption.

That verification mattered because wiping the 1TB drive before confirming the clone worked would leave me with a completely unbootable PC and no working Windows install on either drive. That would have meant a full reinstall from installation media, and losing every program, setting, and any file that wasn't backed up.


## BIOS/UEFI Settings

Before installing Proxmox, I checked the following settings in the UEFI setup on the ASRock board. Everything came back already configured the way Proxmox needed, so no changes required, just confirmation before going ahead with the install.

- **UEFI Boot Mode** (Boot → Boot Mode Select): Controls whether the system boots using modern UEFI firmware or the older Legacy BIOS compatibility mode. Proxmox's installer and boot loader expect UEFI, not Legacy.
- **Intel VT-x** (Advanced → CPU Configuration → Intel Virtualization Technology): Lets the CPU run virtual machines with hardware acceleration instead of slow software emulation. This is the entire reason Proxmox can run VMs at usable speed.
- **Intel VT-d** (Advanced → Chipset Configuration → VT-d): Allows a VM to talk directly to a physical hardware device, like a GPU or USB controller, instead of going through the host. Not needed for this initial build, but useful later for passing real hardware into a VM.
- **AHCI** (Advanced → Storage Configuration → SATA Mode): The standard way the motherboard communicates with SATA/NVMe storage. Proxmox expects this mode rather than the older IDE compatibility mode or RAID mode.
- **SSD Detection**: Confirmed the 1TB Crucial SSD actually showed up on the Storage Configuration screen, by model, before installing anything. If the installer's target disk doesn't match what's physically installed, that points to a seating or slot problem — not something to troubleshoot in software.
- **Hyper-Threading / Integrated Graphics**: Left at their defaults (enabled / automatic), since neither needed to change for this build.


## Installer Choices

I installed Proxmox VE 9.2, the current stable release at the time, using the official ISO downloaded directly from proxmox.com with the checksum verified before use.

For the filesystem, I chose ext4 with LVM-thin rather than ZFS. ZFS is Proxmox's more advanced option, and it adds features like built-in checksumming and snapshots, but it also reserves a meaningful chunk of RAM for its own caching, which isn't something I can afford to give up on a 16GB host that also needs to run multiple VMs. ext4 with LVM-thin is the simpler, lower-overhead choice: it's well-understood, has minimal memory overhead, and is the standard default for a single-disk system like this one. Given the RAM constraint is the main limitation on this build, keeping the storage layer lightweight made more sense than starting with ZFS and fighting for memory from day one.


## Repository Configuration

By default, a fresh Proxmox install points at the Enterprise repository for both core Proxmox packages and Ceph, and that repository requires a paid subscription key to access. Since I don't have a subscription, leaving it enabled meant every update check would fail with an authentication error (401 Unauthorized) before it even got to downloading anything.

I disabled both Enterprise entries (the `pve` one and the `ceph-squid` one, because I'm not using Ceph at all, since that's built for pooling storage across multiple nodes, not a single-node build like this), and added the No-Subscription repository instead. It's the same packages, just without the paid support entitlement, and the standard choice for a home lab. After switching repos, `apt update` and the first full upgrade completed without errors.


## Post-Install Verification

Before considering the host build finished, I ran through a full verification pass rather than just assuming everything worked:

- **Version:** `pveversion` confirmed `pve-manager/9.2.2`, matching the release I installed.
- **CPU / virtualization:** `lscpu` confirmed 1 socket, 6 cores, 2 threads per core (12 CPUs total), matching the i7-8700's actual spec, with VT-x reported as active. (A quicker method I tried first — counting `vmx` flags in `/proc/cpuinfo` — returned 24 instead of 12, which looked wrong at first. `lscpu` gave the cleaner, authoritative answer and confirmed there was no real problem, just an unreliable counting method.)
- **Network:** `ip a` and `ip route` confirmed `vmbr0` held the correct static IP and the default route pointed at the right gateway.
- **Connectivity:** Ping tests to the gateway, to `8.8.8.8`, and to `google.com` all succeeded — confirming local network reachability, internet access, and DNS resolution as three separate, distinct checks rather than assuming one implies the others.
- **RAM and storage:** The Summary page in the web UI confirmed the full 16GB of RAM was recognized, and a S.M.A.R.T. check on the SSD came back PASSED, with both the `local` and `local-lvm` storage pools present and active.

Checking these as separate, individual items, rather than just confirming the web UI loaded, is what actually caught the CPU count anomaly early, even though it turned out to be a false alarm.