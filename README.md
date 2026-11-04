# Homelab Expansion Project: Bare-Metal Hypervisor & Secure Local Infrastructure

I upgraded a Dell OptiPlex and rebuilt it as a single Proxmox VE host, a Type-1 hypervisor that consolidates my standalone machines and runs virtual machines and containers.

---
## Architecture Overview

```text
                          ┌──────────────────────────┐
                          │    LAN Clients / Laptop  │
                          └────────────┬─────────────┘
                                       │
                    HTTPS Requests     │  DNS Lookups (*.homelab.internal)
              ─────────────────────────┼─────────────────────────
             │                         ▼                         │
             │           ┌───────────────────────────┐           │
             │           │  Pi-hole DNS Management   │           │
             │           │  (DNS 53 / Admin 8888)    │           │
             │           └─────────────┬─────────────┘           │
             ▼                         │                         ▼
┌───────────────────────────┐          │          ┌───────────────────────────┐
│    Nginx Proxy Manager    │◄─────────┘          │        Proxmox VE         │
│  (Central SSL / Edge TLS) │                     │   (Type-1 Hypervisor Host)│
└────────────┬──────────────┘                     └─────────────┬─────────────┘
             │                                                  │
             │ Reverse Proxy (HTTPS + WebSockets)               │
             └──────────────────────────────────────────────────┘
                                       │
                                       ▼
                       ┌───────────────────────────────┐
                       │  Management GUI (Port 8006)   │
                       └───────────────────────────────┘
```


---

## Technical Specifications

### 🖥️ Compute & Memory
*   **CPU:** Intel Core i7-4790 (4 cores, 8 threads, 3.60 GHz base / 4.00 GHz turbo)
*   **Memory:** 32 GB dual-channel DDR3 (4 x 8 GB, non-ECC)
*   **Graphics:** AMD Radeon Pro WX 5100 (50W peak draw from the PCIe slot)
*   **Chassis & Power:** Dell OptiPlex 7020 MT with the stock 290W power supply

### 💾 Storage
*   **Operating System Drive:** 1 x 250 GB Patriot SATA SSD on motherboard port SATA-0
*   **Data Drives:** 4 x hard drives (3 x 2.5-inch laptop drives and 1 x shucked 3.5-inch desktop drive)
*   **Storage Controller:** ASMedia ASM1166 6-port SATA expansion card in a physical x16 slot wired for x4
*   **Storage Pool:** **ZFS RAIDZ1** with about 1.5 TB of storage, single-disk parity, and checksums that can detect and repair data errors

### 🌐 Network & Services
*   **Networking:** Static IP (`<host-IP>/24`) on the onboard Intel Gigabit Ethernet port
*   **Local Domain:** `homelab.internal`
*   **DNS Tier:** Containerized Pi-hole serving DNS on port `53`, with its web admin on `8888`, using a `dnsmasq` wildcard for `*.homelab.internal`.
*   **Reverse Proxy:** Nginx Proxy Manager routes traffic, serves the internal SSL certificates, and passes the WebSocket traffic that the `noVNC` console needs.

---

## Step-by-Step Implementation

### Phase 1: Hardware Upgrades & Installation
1.  **Drive Bay Space:** Confirmed that one of the optical drive bays was free, so I could use the upper 5.25" bay for drives.
2.  **CPU & Memory:** Upgraded to an Intel i7-4790 with fresh thermal paste, and expanded to 32 GB of RAM using the motherboard's color-coded memory channels.
3.  **GPU & Storage Controller:** Installed the AMD Radeon Pro WX 5100 in the primary PCIe slot, replacing the stock Radeon GPU. Installed the 6-port ASM1166 controller in the secondary x16 slot, which is wired for x4.
4.  **Drive Mounting & Cabling:**
    *   Used a Corsair CSSD-BRKT2 dual-SSD bracket to mount the 250 GB Patriot boot SSD and one 2.5" laptop drive.
    *   Mounted the 3.5" desktop drive in the lower drive cage, using the factory blue tray.
    *   Fastened the other two 2.5" laptop drives into one bracket, insulated their exposed circuit boards, and secured the assembly in the upper 5.25" bay.
5.  **Power Layout:**
    *   **Line A (direct):** Gave the shucked 3.5" drive its own unsplit power line from the 290W power supply to handle its 12V startup surge.
    *   **Lines B & C (split):** Ran the three low-power laptop drives and the boot SSD from the remaining leads using SATA power Y-splitters.
6.  **Pre-Install Testing:** Ran Dell's built-in ePSA diagnostics before connecting the data cables, so I could confirm the core hardware was stable before adding the storage.

### Phase 2: Firmware Settings
1.  **Boot Loop Fix:** Fixed a boot loop caused by an old Linux Mint installation on a reused drive taking over the motherboard's default boot path.
2.  **BIOS Settings:** Enabled Secure Boot, turned off Intel RST to run the drives in **AHCI mode**, and enabled Intel Virtualization Technology (**VT-x**) and Directed I/O (**VT-d**).
3.  **Boot Order:** Set the system to boot in UEFI mode from the Patriot SSD.

### Phase 3: Proxmox VE Installation
1.  **Installer:** Wrote the latest Proxmox VE installer to a USB 3.0 drive with Rufus, using a UEFI partition layout.
2.  **Network Settings:** Set the network configuration in the installer before connecting the machine to the network:
    *   **FQDN:** `pve01.homelab.internal`
    *   **Management IP:** `<host-IP>/24`
    *   **Gateway:** `<gateway-IP>`
    *   **DNS:** My home network's DNS at first, changed to Pi-hole later

### Phase 4: ZFS Storage Pool
1.  **Controller Check:** Confirmed that the Linux kernel recognized the new PCIe controller. `dmesg | grep -i ahci` showed it mapped to SCSI hosts 6 through 37, which is 32 logical ports.
2.  **Disk Wipe:** Used `lsblk` in the Proxmox shell to identify the four hard drives (`/dev/sdb`, `/dev/sdc`, etc.), then wiped their old filesystems and the leftover Linux Mint partitions.
3.  **Pool Creation:** Created a ZFS pool from the four clean drives using single-disk parity (`raidz`), giving about 1.5 TB of fault-tolerant storage for application data and VM disks.

### Phase 5: DNS, Proxy & TLS
1.  **DNS:** Pointed the domain at the Pi-hole container, and verified that the `dnsmasq` wildcard sent any `*.homelab.internal` name to Nginx Proxy Manager without separate A records.
2.  **Proxy Host:** Created a proxy host in Nginx Proxy Manager for `pve01.homelab.internal`.
3.  **Proxy Settings:** Configured the connection to the Proxmox web interface:
    *   **Forward Scheme:** Set to `HTTPS`, because Proxmox serves its interface over HTTPS on port `8006`.
    *   **WebSocket Support:** Enabled so that noVNC consoles, terminals, and live metrics don't keep disconnecting.
    *   **SSL:** Applied the trusted local certificates to the proxy host, so browsers on the network show a valid connection.

---

## Verification & Status

*   **Compute:** All 8 threads are online, and idle temperatures are within normal limits.
*   **Storage:** The ZFS pool initialized cleanly. Periodic ZFS scrubs check data integrity. Any single drive can fail without taking the data offline.
*   **Networking:** Opening `https://homelab.internal` connects through Nginx Proxy Manager, validates the local SSL certificate, and reaches the Proxmox dashboard over a single secure port.

---

## Future Roadmap
*   [ ] Deploy lightweight Ubuntu Minimal or Debian cloud-init images.
*   [ ] Build a multi-node Kubernetes cluster.
*   [ ] Set up automated persistent volume (PV) claims on the 1.5 TB ZFS pool.
