# Setup Guide — Active Directory Lab on Proxmox

A step-by-step, reproducible walkthrough for building the Windows Server 2022 domain
controller and joining a client, on a Proxmox host.

---

## Prerequisites

- A running **Proxmox VE** host with internet access and enough resources
  (~2 GB for the host + 5 GB for the DC + 4 GB per client).
- ISOs uploaded to Proxmox:
  - **Windows Server 2022** evaluation (Microsoft Evaluation Center — 180-day, free)
  - **Windows 11** (Microsoft "Download Windows 11 Disk Image (ISO)")
  - Picking static IP address for the DC

---

## Part 1 — Create the Domain Controller VM

In the Proxmox UI, I created a VM and set it to these values:

- General | Name | `DC01` 
- OS | ISO | Windows Server 2022 
- OS | Type / Version | Microsoft Windows / 11-2022 
- System | Machine / BIOS | `i440fx` / `SeaBIOS` 
- System | QEMU Agent | enabled 
- Disks | Size / Bus | `60 GB` / SATA
- CPU | Cores / Type | `2` / `host` 
- Memory | RAM | `5120 MB` 
- Network | Bridge / Model | `vmbr0` / Intel E1000 

<img width="715" height="538" alt="Creating Virtual Machine" src="https://github.com/user-attachments/assets/e3403aa1-6cee-4ee8-869e-0e3cb0f0f308" />


---

## Part 2 — Install Windows Server 2022

1. Boot from the ISO 
2. Set language/keyboard 
3. Edition: **Windows Server 2022 Standard** 

<img width="1025" height="822" alt="Windows Version Selection" src="https://github.com/user-attachments/assets/05c5e828-a6d8-45ef-b27d-64819613ea43" />


7. At first boot, I set the **Administrator** password 
8. Logging in: in the Proxmox console, I used the **Ctrl+Alt+Del** toolbar button, then my password.
9. I was greeted by Windows Server Manager automatically opening.

<img width="1016" height="825" alt="Windows Server Manager Open" src="https://github.com/user-attachments/assets/f92f7209-1912-4954-912e-52c3d3e69553" />



---

## Part 3 — Base configuration

**Rename the computer**:
I renamed the computer to "DC01"

1. Server Manager - Local Server - click the computer name.
2. Change - set name to `DC01` - OK → restart.

**Set a static IP with self-referencing DNS.** Open network adapter settings
(`Win + R` → `ncpa.cpl` → adapter → **Properties** → **Internet Protocol Version 4** →
**Properties**) and set

- The DC's DNS must point at its own IP. It will be the domain's DNS server.


Verify with:
```
ipconfig /all
```
Confirm the IPv4 address is correct, `DHCP Enabled: No`, and `DNS Servers` shows the DC's
own IP.

---

## Part 4 — Install AD DS and promote to Domain Controller

**Install the role:**
1. Server Manager → **Manage** → **Add Roles and Features**.
2. Installation type: *Role-based or feature-based* → your server → **Server Roles**.
3. Check **Active Directory Domain Services** → **Add Features** when prompted.
4. Next through the screens → **Install**.

<img width="773" height="554" alt="Installing" src="https://github.com/user-attachments/assets/42423f36-d9c4-43e2-9187-3981d011410f" />


**Promote the server:**
1. Click the yellow flag → **Promote this server to a domain controller**.
2. **Add a new forest** → Root domain name: `lab.local`.

<img width="689" height="332" alt="Promoting to DC" src="https://github.com/user-attachments/assets/5a83f20c-6571-4404-80e5-8ee20ff94a08" />


4. Domain Controller Options:
   - Leave functional levels at default (Windows Server 2016 is the max on 2022).
   - Keep **DNS server** checked.
   - Set the **DSRM** (Directory Services Restore Mode) recovery password.
5. **DNS Options:** ignore the delegation warning (expected for a standalone forest).
6. **NetBIOS name:** confirm the auto-filled value (e.g. `LAB`).
7. Paths → defaults → **Review** → **Prerequisites Check** → **Install**.

The server reboots. Logins now read `LAB\Administrator` — the domain is live.

---

## Part 6 — Create directory objects

In **Active Directory Users and Computers (ADUC)** (Tools → ADUC):

1. **OU:** right-click `lab.local` → **New → Organizational Unit** → `Lab Users`.

<img width="307" height="390" alt="Organizational Unit Created" src="https://github.com/user-attachments/assets/917b5843-e3d3-45d4-9c1d-c70d315dbddb" />


2. **User:** right-click `Lab Users` → **New → User** →
   - Name: `Test User`, logon: `testuser`
   - Set a compliant password (must not contain the username)
   - Uncheck *"must change password at next logon"*; check *"password never expires"*

<img width="428" height="370" alt="New User Creation" src="https://github.com/user-attachments/assets/54abcce2-06f8-4c08-b6fe-60c72faf80c5" />

    
3. **Group:** right-click `Lab Users` → **New → Group** → `Lab Team` (Global / Security).
4. **Membership:** double-click `Lab Team` → **Members** → **Add** → `testuser` →
   **Check Names** → OK.

<img width="750" height="517" alt="Adding New User to Group" src="https://github.com/user-attachments/assets/17b11fc2-3458-4166-89ef-eadd4da2550b" />


---

## Part 7 — Build the first client (in progress)

Create `WIN11-A` in Proxmox. Windows 11 requires UEFI + Secure Boot + TPM, so this VM
differs from the DC:


- General | Name | `WIN11-A` |
- OS | ISO / Version | Windows 11 / 11-2022 
- System | Machine | `q35` 
- System | BIOS | `OVMF (UEFI)` + add EFI disk 
- System | TPM | Add TPM State, v2.0
- Disks | Size / Bus | `64 GB` / SATA 
- CPU | Cores / Type | `2` / `host` 
- Memory | RAM | `4096 MB` 
- Network | Bridge / Model | `vmbr0` / Intel E1000 

**Install:** boot the ISO - *"I don't have a product key"* - **Windows 11 Pro**
 - Custom install → 64 GB disk.

**Local account bypass** (Win11 pushes a Microsoft account; a domain client wants a local one):
1. At the network/sign-in screen press **Shift + F10** for a command prompt.
2. Run:
   ```
   OOBE\BYPASSNRO
   ```
3. The VM reboots; choose "I don't have internet" - "Continue with limited setup".
4. Create a local account with a password you record.


---

## Troubleshooting notes


- Windows can't see the disk at install | Disk bus is VirtIO without driver — use SATA, or load virtio-win. 
- Password rejected creating a user | Complexity rule; also can't contain the username. 
- Proxmox memory graph looks maxed | Windows caches free RAM — check the guest's Task Manager for real usage. 
- Client won't join the domain | Client DNS not pointing at the DC — fix Preferred DNS to the DC's IP. 
- Static-IP warning during promotion | Usually IPv6 still on auto; safe if IPv4 is static. 
