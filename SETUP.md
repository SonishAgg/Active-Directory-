# Setup Guide — Active Directory Lab on Proxmox

A step-by-step, reproducible walkthrough for building the Windows Server 2022 domain
controller and joining a client, on a Proxmox host.

> **Scope:** Covers everything through the domain controller being live and populated, plus
> the start of the first client. Client join and beyond are marked 🚧 and will be completed
> in a later revision.
>
> **Conventions:** Passwords are never shown — use your own and record them in a local
> credentials file (not committed). IPs use an example `192.168.1.0/24` scheme; substitute
> your own network. The DC is `192.168.1.10`, the router/gateway `192.168.1.1`.

---

## Prerequisites

- A running **Proxmox VE** host with internet access and enough resources
  (~2 GB for the host + 5 GB for the DC + 4 GB per client).
- ISOs uploaded to Proxmox (**Datacenter → your node → local → ISO Images → Upload**):
  - **Windows Server 2022** evaluation (Microsoft Evaluation Center — 180-day, free)
  - **Windows 11** (Microsoft "Download Windows 11 Disk Image (ISO)")
- Decide your addressing before starting. Pick a static IP for the DC that sits **outside**
  your router's DHCP range so it can't collide with a leased address.

---

## Part 1 — Create the Domain Controller VM

In the Proxmox UI, click **Create VM** and set:

| Tab | Setting | Value |
|---|---|---|
| General | Name | `DC01` |
| OS | ISO | Windows Server 2022 |
| OS | Type / Version | Microsoft Windows / 11-2022 |
| System | Machine / BIOS | `i440fx` / `SeaBIOS` |
| System | QEMU Agent | ✅ enabled |
| Disks | Size / Bus | `60 GB` / **SATA** |
| CPU | Cores / Type | `2` / `host` |
| Memory | RAM | `5120 MB` |
| Network | Bridge / Model | `vmbr0` / Intel E1000 |

> **Why SATA:** Windows detects a SATA disk during install with no extra driver. VirtIO is
> faster but requires loading a driver mid-install.
> **Why CPU `host`:** full performance and CPU feature pass-through.

Finish, then **Start** the VM and open the **Console**.

---

## Part 2 — Install Windows Server 2022

1. Boot from the ISO (press a key at *"Press any key to boot from CD/DVD"* if prompted).
2. Set language/keyboard → **Install now**.
3. Edition: **Windows Server 2022 Standard (Desktop Experience)** — the GUI version.
4. Accept the license → **Custom: Install Windows only (advanced)**.
5. Select the 60 GB disk → **Next**. Windows installs and reboots itself.
6. At first boot, set the **Administrator** password (meets complexity: 3 of
   upper/lower/number/symbol).
7. Log in: in the Proxmox console, use the **Ctrl+Alt+Del** toolbar button, then your password.

---

## Part 3 — Base configuration

**Rename the computer** (do this before promotion):
1. Server Manager → **Local Server** → click the computer name.
2. **Change…** → set name to `DC01` → OK → restart.

**Set a static IP with self-referencing DNS.** Open network adapter settings
(`Win + R` → `ncpa.cpl` → adapter → **Properties** → **Internet Protocol Version 4** →
**Properties**) and set:

```
IP address:        192.168.1.10
Subnet mask:       255.255.255.0
Default gateway:   192.168.1.1
Preferred DNS:     192.168.1.10   ← the DC points at ITSELF
```

> **The DC's DNS must point at its own IP.** It is about to become the domain's DNS server.
> Pointing DNS at the router or a public resolver (e.g. 8.8.8.8) causes promotion errors.

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

**Promote the server:**
1. Click the yellow ⚠ flag → **Promote this server to a domain controller**.
2. **Add a new forest** → Root domain name: `lab.local`.
3. Domain Controller Options:
   - Leave functional levels at default (Windows Server 2016 is the max on 2022).
   - Keep **DNS server** checked.
   - Set the **DSRM** (Directory Services Restore Mode) recovery password.
4. **DNS Options:** ignore the delegation warning (expected for a standalone forest).
5. **NetBIOS name:** confirm the auto-filled value (e.g. `LAB`).
6. Paths → defaults → **Review** → **Prerequisites Check** → **Install**.

The server reboots. Logins now read `LAB\Administrator` — the domain is live.

> A *"network adapter does not have a static IP"* warning at promotion is usually just IPv6
> (still automatic); safe to proceed as long as your IPv4 is static.

---

## Part 5 — Verify

Run in Command Prompt / PowerShell:
```powershell
dcdiag
nltest /dsgetdc:lab.local
```
`dcdiag` should report passing tests (a couple of benign warnings are normal on a fresh
single-DC lab). Server Manager should now list **AD DS** and **DNS**, and **Tools** should
include Active Directory Users and Computers, Group Policy Management, and DNS.

---

## Part 6 — Create directory objects

In **Active Directory Users and Computers (ADUC)** (Tools → ADUC):

1. **OU:** right-click `lab.local` → **New → Organizational Unit** → `Lab Users`.
2. **User:** right-click `Lab Users` → **New → User** →
   - Name: `Test User`, logon: `testuser`
   - Set a compliant password (must not contain the username)
   - Uncheck *"must change password at next logon"*; check *"password never expires"*
3. **Group:** right-click `Lab Users` → **New → Group** → `Lab Team` (Global / Security).
4. **Membership:** double-click `Lab Team` → **Members** → **Add** → `testuser` →
   **Check Names** → OK.

---

## Part 7 — Build the first client 🚧 *(in progress)*

Create `WIN11-A` in Proxmox. Windows 11 requires UEFI + Secure Boot + TPM, so this VM
differs from the DC:

| Tab | Setting | Value |
|---|---|---|
| General | Name | `WIN11-A` |
| OS | ISO / Version | Windows 11 / 11-2022 |
| System | Machine | `q35` |
| System | BIOS | `OVMF (UEFI)` + add EFI disk |
| System | TPM | Add TPM State, **v2.0** |
| Disks | Size / Bus | `64 GB` / SATA |
| CPU | Cores / Type | `2` / `host` |
| Memory | RAM | `4096 MB` |
| Network | Bridge / Model | `vmbr0` / Intel E1000 |

**Install:** boot the ISO → *"I don't have a product key"* → **Windows 11 Pro**
(Home cannot join a domain) → Custom install → 64 GB disk.

**Local account bypass** (Win11 pushes a Microsoft account; a domain client wants a local one):
1. At the network/sign-in screen press **Shift + F10** for a command prompt.
2. Run:
   ```
   OOBE\BYPASSNRO
   ```
3. The VM reboots; choose **"I don't have internet"** → **"Continue with limited setup"**.
4. Create a local account (e.g. `labadmin`) with a password you record.

### 🚧 Remaining steps (to be documented)
- Set the client's static IP (`192.168.1.11`) with **Preferred DNS = the DC** (`192.168.1.10`).
  *(Client DNS points at the DC, not at itself — only the DC self-references.)*
- Join `WIN11-A` to `lab.local` (System → Domain or workgroup → join), reboot.
- Log in as `LAB\testuser` to confirm end-to-end authentication.
- Clone `WIN11-A` → `WIN11-B`, then rename and re-IP.

---

## Troubleshooting notes

| Symptom | Cause / fix |
|---|---|
| Windows can't see the disk at install | Disk bus is VirtIO without driver — use SATA, or load virtio-win. |
| Password rejected creating a user | Complexity rule; also can't contain the username. |
| Proxmox memory graph looks maxed | Windows caches free RAM — check the guest's Task Manager for real usage. |
| Client won't join the domain | Client DNS not pointing at the DC — fix Preferred DNS to the DC's IP. |
| Static-IP warning during promotion | Usually IPv6 still on auto; safe if IPv4 is static. |
