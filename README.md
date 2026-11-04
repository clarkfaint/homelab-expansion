# Homelab Expansion Project: Bare-Metal Hypervisor & Secure Local Infrastructure

A comprehensive, production-grade overhaul of local computing, storage, and networking layers to establish a highly available, secure, and monitored Type-1 Hypervisor node. This project involved migrating standalone legacy workstations into a unified, enterprise-grade bare-metal infrastructure deployment optimized for virtualized cluster scaling and containerized microservices.

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

### 🖥️ Compute & Memory Nodes
*   **Host CPU:** Intel Core i7-4790 (4 Cores, 8 Threads @ 3.60 GHz / 4.00 GHz Turbo)
*   **Memory Array:** 32GB Dual-Channel DDR3 Configuration (4 x 8GB Non-ECC Modules)
*   **Dedicated Graphics:** AMD Radeon Pro WX 5100 Workstation GPU (50W Peak Slot-Draw)
*   **Chassis & Power System:** Dell OptiPlex 7020 MT Architecture backed by an OEM 290W Power Supply unit, carefully balanced for peak inrush startup current margins.

### 💾 Storage Topology
*   **Operating System Volume:** 1 x 250GB Patriot SATA Solid State Drive (Isolated natively on Motherboard SATA-0 controller link).
*   **Data Pool Array:** 4 x Mechanical Hard Disk Drives (Unified multi-form factor matrix combining 3 x 2.5-inch thin laptop drives and 1 x shucked 3.5-inch high-capacity desktop enclosure drive).
*   **Storage Controller:** ASMedia ASM1166 6-Port PCIe x4 SATA Expansion Card seated into an electronically wired x4 mechanical x16 PCIe lane.
*   **Storage Pool Architecture:** Software-Defined **ZFS File System (RAIDZ1 Configuration)** providing ~1.5 TB of storage capacity with dedicated single-disk parity fault tolerance and data self-healing metadata verification.

### 🌐 Network Infrastructure & Services
*   **Base Networking:** Hardcoded static local IP mapping (`192.168.68.68/24`) tied natively to the onboard Intel Gigabit Ethernet hardware stack.
*   **Domain Management:** Segmented local resolution landscape running under the private enterprise domain: `homelab.internal`.
*   **DNS Tier:** Containerized Pi-hole serving DNS on port `53`, with its web admin on `8888`, using a `dnsmasq` wildcard for `*.homelab.internal`.
*   **Layer 7 Traffic Delivery:** Nginx Proxy Manager (NPM) acting as a secure edge proxy mesh. Orchestrates programmatic traffic routing, unifies internal custom SSL certificate bindings, and explicitly handles persistent Layer 7 connection integrity via custom WebSocket upstream streaming blocks for virtual console access (`noVNC`).

---

## Step-by-Step Implementation Documentation

### Phase 1: Hardware Engineering & Component Optimization
1.  **Chassis Demolition & Clearance:** Confirmed clearance in one of the obsolete optical drive arrays to utilize physical chassis space inside the upper 5.25" media bay.
2.  **Silicon Lifecycle Upgrade:** Upgraded the legacy processing core to an Intel i7-4790 processor, executing fresh thermal paste applications and performing full-capacity expansion to 32GB RAM utilizing color-coded motherboard memory bank channels.
3.  **Storage Controller & GPU Integration:** Seated the workstation AMD Radeon Pro WX 5100 GPU into the primary PCIe slot (replacing OEM Radeon GPU). Integrated the unbranded 6-Port ASM1166 PCIe controller into the secondary mechanical x16 (wired x4) slot. 
4.  **Physical Drive Arrays & Cable Management:**
    *   Utilized a Corsair CSSD-BRKT2 dual-SSD bracket to mount the 250GB Patriot Boot SSD alongside a 2.5" laptop HDD using standard side alignment casing mounts.
    *   Mounted the 3.5" desktop drive into the secondary lower drive cage track inside the primary factory blue drive tray caddy.
    *   Hard-mounted the remaining 2 x 2.5" laptop drives into a unified utility alignment bracket, safely insulating the lower exposed PCBs, and precision-secured the cluster assembly securely within the vacated 5.25" upper media bay.
5.  **Power Balancing & Electrical Layout:**
    *   **Line A (Direct Rail):** Dedicated a primary, un-split native power line straight from the 290W PSU to the shucked 3.5" drive to comfortably shield the system from massive 12V inrush startup spin-up currents.
    *   **Line B & C (Splitter Architecture):** Distributed the remaining 3 x low-draw laptop HDDs and the boot SSD across secondary power leads utilizing balanced high-grade SATA power Y-splitters.
6.  **Pre-Flight Verification:** Executed standalone hardware initialization testing before linking physical data cables, running full Dell Embedded System Assessments (ePSA) to isolate core compute stability from storage dependencies.

### Phase 2: Firmware & Low-Level Logical Adjustments
1.  **UEFI Path Optimization:** Remedied an internal boot-loop anomaly where the motherboard default boot track was hijacked by an old Linux Mint installation resting on a repurposed storage disk. 
2.  **BIOS Standardization:** Booted into system parameters, enabled Secure Boot configurations, turned off Intel RST to lock down true raw **AHCI Mode** operation for optimal kernel visualization, and explicitly enabled Intel Virtualization Technology (**VT-x**) and Directed I/O (**VT-d**).
3.  **Boot Priority Lock:** Forced the system to boot explicitly in native UEFI mode, mapping the primary execution path directly onto the target Patriot Boot SSD volume.

### Phase 3: Operating System Deployment (Proxmox VE)
1.  **Base System Flashing:** Deployed the latest enterprise Proxmox VE image utilizing a UEFI-partitioned Rufus layout on USB 3.0 interfaces.
2.  **Offline Static Configuration:** Pre-assigned and hardcoded all critical networking layers directly into the installer environment before physical network mapping:
    *   **FQDN:** `pve01.homelab.internal`
    *   **Management IP:** `192.168.68.68/24`
    *   **Gateway:** `192.168.68.1`
    *   **DNS:** Primary home routing pathways (later transitioned downstream to Pi-hole loopsbacks).

### Phase 4: Software-Defined Storage Pool Creation
1.  **Kernel Host Verification:** Verified that the Linux kernel successfully recognized the new PCIe controller. Running `dmesg | grep -i ahci` confirmed that the underlying controller properly mapped SCSI hosts 6 through 37, opening 32 distinct logical channels via port multiplier visibility.
2.  **Data Purge Processing:** Accessed the Proxmox terminal shell interface, targeting all four attached raw mechanical storage blocks seen via `lsblk` (`/dev/sdb`, `/dev/sdc`, etc.). Used the disk subsystem configurations to wipe old filesystems, permanently scrubbing the legacy Linux Mint partition flags.
3.  **ZFS Deployment:** Grouped the four completely clean mechanical storage disks into a software-defined ZFS layout. Selected the single-disk parity (`raidz`) architecture to establish a secure, fault-tolerant ~1.5 TB volume block optimized for persistent application volumes and VM disks.

### Phase 5: Traffic Management, Routing, & Security Architecture
1.  **DNS Overlay:** Linked the domain mapping to the network's Pi-hole container. Verified that the custom `dnsmasq` wildcard setup seamlessly routed any traffic addressed to `*.homelab.internal` straight to the Nginx Proxy Manager server without requiring separate standalone local A-records.
2.  **Layer 7 Traffic Proxy Tuning:** Constructed a new proxy host gateway card inside Nginx Proxy Manager mapped directly to `pve01.homelab.internal`.
3.  **Upstream Stabilization Rules:** Adjusted Nginx configuration parameters to correctly pass SSL handshakes down to Proxmox's native backend:
    *   **Forward Scheme:** Set strictly to `HTTPS` to match Proxmox’s internal forced port `8006` security engine rules.
    *   **WebSocket Upstream Support:** Explicitly enabled to prevent remote noVNC shell consoles, terminal frameworks, and live metrics from suffering constant micro-disconnect drop loops.
    *   **SSL Centralization:** Bound trusted local security certificates directly to the routing path, successfully outputting a full green browser validation lock across the network landscape.

---

## Verification & Status

*   **Compute Status:** 8 Logical threads fully online, idle temperatures maintaining standard limits under fresh thermal design integration.
*   **Storage Health:** ZFS pool initialized cleanly. Continuous ZFS background scrubs active. Any single drive can fail completely without halting system data availability.
*   **Networking Verification:** Accessing `https://homelab.internal` instantly handshakes with the NPM proxy layer, verifies local SSL certificate authority bindings, and routes traffic cleanly down to the hypervisor dashboard over a single secure port mapping.

---

## Future Roadmap
*   [ ] Deploy lightweight Ubuntu Minimal/Debian cloud-init image configurations.
*   [ ] Initialize a high-performance bare-metal multi-node Kubernetes cluster footprint.
*   [ ] Implement automated persistent volume (PV) storage claims bound directly to the 1.5 TB ZFS data pool tier.
