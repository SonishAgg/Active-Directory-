# Active Directory Home Lab — Windows Server 2022 Domain Controller on Proxmox

> **Status:** Work in progress — core domain controller is live and verified; first client is mid-build.

A self-hosted Active Directory lab built on a Proxmox homelab. The goal is hands-on
experience with enterprise identity infrastructure — standing up a domain controller,
running DNS, managing users/groups/OUs, and joining client machines to the domain — in a
safe, self-contained environment.

This project is part of a broader homelab and self-hosting learning track.

---

## Why this project

Active Directory is the backbone of the majority of enterprise networks, which
makes it foundational for sysadmin work and one of the most common attack surfaces in
security. Building one end-to-end, rather than reading about it, means understanding how
authentication, DNS, and policy actually fit together.

**Skills demonstrated:**
- Virtualization on Proxmox (VM provisioning, hardware config, networking)
- Windows Server 2022 installation and base configuration
- Active Directory Domain Services (AD DS) deployment
- DNS fundamentals (self-referencing DNS, forest DNS zones)
- Directory administration (OUs, users, groups)
- Static IP addressing and network design decisions
- Documentation and reproducibility

---

## Tech Stack

**Proxmox host** — hypervisor running the whole lab
- **Network**: single bridge vmbr0, static IPs, no DHCP
- **DC01** - Windows Server 2022
-     Role: Domain Controller
-     Runs AD DS and DNS
-     Domain: lab.local
-     Static IP, DNS points to itself
- **WIN11-A** - Windows 11 Pro (in-progress)
-     Role: Domain Client
-     Member of lab.local
-     Static IP, DNS points to DC01

---

## Environment

### Proxmox host
- Proxmox VE, Intel business workstation (4-core, Haswell-era), enterprise SSD storage.

### DC01 — Domain Controller VM

- OS | Windows Server 2022 Standard (Desktop Experience)
- Machine / BIOS | i440fx / SeaBIOS 
- Disk | 60 GB, SATA bus (no driver needed at install) 
- CPU | 2 cores, type `host` 
- Memory | 5 GB (4 GB minimum; 5 GB comfortable) 
- Network | `vmbr0`, Intel E1000 model 
- QEMU Agent | Enabled (channel only; guest agent install pending)

### WIN11-A — Client VM (in progress)

- OS | Windows 11 Pro 
- Machine / BIOS | q35 / OVMF (UEFI) + EFI disk 
- Security | TPM 2.0 device, Secure Boot 
- Disk | 64 GB, SATA bus 
- CPU | 2 cores, type `host` 
- Memory | 4 GB 
- Network | `vmbr0`, Intel E1000 model 

---

## Roadmap

- [x] Deploy Windows Server 2022 domain controller (`lab.local`)
- [x] Verify with `dcdiag`
- [x] Create OU, user, and group
- [ ] Finish Windows 11 client install (local account)
- [ ] Set client static IP with DNS pointed at the DC
- [ ] Join `WIN11-A` to the domain and log in as `LAB\testuser`
- [ ] Clone `WIN11-A` → `WIN11-B` (second client)
- [ ] Group Policy experiments (password policy, drive mapping, desktop lockdown)
- [ ] (Stretch) AD security exploration in the isolated segment


