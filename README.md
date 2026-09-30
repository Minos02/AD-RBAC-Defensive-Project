<div align="center">

# 🛡️ Active Directory Defensive Project
### Role-Based Access Control (RBAC) via Group Policy Objects

![Windows Server](https://img.shields.io/badge/Windows%20Server-2022-0078D6?logo=windows&logoColor=white)
![Active Directory](https://img.shields.io/badge/Active%20Directory-GPO-blue?logo=microsoft&logoColor=white)
![UTM](https://img.shields.io/badge/Virtualization-UTM%20(Apple%20Silicon)-lightgrey)
![Status](https://img.shields.io/badge/Status-RBAC%20Complete-success)

</div>

---

## 📋 Overview

This project implements **defense-in-depth access control** inside a simulated corporate Active Directory environment. Using **Organizational Units (OUs)**, **Security Groups**, and **Group Policy Objects (GPOs)**, three departments — HR, IT, and Finance — are each given a distinct, least-privilege security posture enforced entirely at the domain level.

Built as part of the **Computer Network Defense (CND)** coursework, this lab demonstrates real blue-team hardening techniques: restricting tool access by role, locking down removable storage, enforcing stricter password policies, and verifying enforcement through live testing and Windows' built-in policy diagnostic tools.

## 🧪 Lab Environment

| Component | Details |
|---|---|
| **Domain Controller** | Windows Server 2022 |
| **Domain Name** | `Company.local` |
| **Client (GPO testing)** | Windows 11 |
| **Client (legacy testing)** | Windows 7 |
| **Virtualization** | UTM (macOS host, Apple Silicon) |
| **AD Tools** | Active Directory Users and Computers (`dsa.msc`), Group Policy Management (`gpmc.msc`) |

## 🏢 Organizational Structure

Three OUs were created directly under the domain root, each mapped to a real-world department:

| OU | Purpose |
|---|---|
| **HR** | Restricted tooling, normal desktop use |
| **IT** | Elevated administrative tool access |
| **Finance** | Locked-down USB & Control Panel, strict password policy |

Each OU has a matching **Global Security Group** (`HR_Group`, `IT_Group`, `Finance_Group`) for future permission and reporting flexibility.

## ⚙️ Group Policy Objects

| GPO | Linked OU | What it enforces |
|---|---|---|
| **HR Policy** | HR | Blocks Command Prompt & Registry Editor; normal desktop access otherwise |
| **IT Policy** | IT | Explicitly *allows* Command Prompt & admin tools — least-privilege doesn't mean *no* privilege |
| **Finance Policy** | Finance | Blocks all removable storage (USB/SD/HDD) & Control Panel; stronger password policy (12+ chars, complexity enforced) |

## ✅ Verification & Testing

Every rule was tested **live** on a domain-joined Windows 11 client, not just configured and assumed working:

- 🔒 **HR user** → Command Prompt blocked with a policy-restriction popup
- 🛠️ **IT user** → Command Prompt & admin tools work fine; confirmed via `whoami`
- 🚫 **Finance user** → Control Panel and external storage devices blocked
- 📜 **`gpresult /r`** → confirms exactly which GPO applied to each session
- 🔁 **OU-move test** → moving a user from HR → IT instantly changed their applied policy from *HR Policy* to *IT Policy*, proving enforcement follows **OU location**, not the user account itself
- 🌳 **Policy inheritance** → verified via the Group Policy Inheritance tab, confirming no unwanted conflicts between OU-level and domain-level policy

## 🍏 Building the Lab on Apple Silicon (M-series Mac)

Most Active Directory home labs assume an Intel/x86 machine with VMware or VirtualBox. Running this entirely on an **Apple Silicon Mac** meant there was no official Windows Server ARM ISO to fall back on, so the whole environment had to be built and debugged from scratch using **UTM** as the hypervisor:

- Windows Server 2022 (DC) and Windows 11/7 clients were configured as UTM virtual machines on ARM hardware, working around the lack of native x86 support that most AD tutorials assume
- Networking between VMs had to be manually aligned — UTM defaults to giving VMs separate virtual adapters, which silently breaks domain communication if the DC and client aren't bridged onto the *same* virtual network
- This project's real troubleshooting work (below) largely stems from Apple Silicon virtualization quirks rather than AD misconfiguration alone — proof that the lab was built and debugged first-hand, not just followed from a guide

## 🔧 Troubleshooting Highlights

Domain connectivity broke completely at one point with a **"domain isn't available"** login error. Root-caused and fixed as follows:

**Environment at time of issue**
| Item | Value |
|---|---|
| Domain | `Company.local` (NetBIOS: `COMPANY`) |
| Domain Controller | `DC1` — `192.168.68.108` |
| Windows 11 Client | `WIN11-CLIENT1` — `192.168.68.102` |

**Root cause:** the client was pointed at the wrong DNS server (`192.168.64.10`, from UTM's second virtual adapter) instead of the actual Domain Controller (`192.168.68.108`) — so the client could never resolve or reach the domain at all.

**Fix applied:**
```powershell
netsh interface ipv4 set dnsservers name="Ethernet" static 192.168.68.108 primary
ipconfig /registerdns
```
A stale DNS record pointing the client to an old, incorrect IP was also found and removed directly from the `Company.local` DNS zone.

**Verification commands used:**
```powershell
nltest /dsgetdc:Company.local      # confirmed DC1 was located correctly
nltest /sc_verify:Company.local    # confirmed secure channel trust (NERR_Success)
gpupdate /force                    # re-applied Group Policy after the fix
gpresult /r                        # confirmed the correct GPO was applied
```

**Result:** once DNS was corrected, `nltest` confirmed the secure channel trust, Group Policy applied successfully, and the intended Control Panel restriction was verified live on the client. One known limitation: `gpresult /h` (HTML export) returned Access Denied even from an elevated prompt — but since `gpresult /r` fully confirmed policy application, this didn't block validating the lab.

| Check | Status |
|---|---|
| Domain connectivity | ✅ Working |
| DNS resolution | ✅ Working |
| AD secure channel / trust | ✅ Working |
| Group Policy refresh | ✅ Working |
| IT Policy application | ✅ Verified |
| `gpresult /h` HTML export | ⚠️ Access Denied (not required for validation) |

## 📄 Full Report

The complete project write-up — including all screenshots, exact GPO paths, and step-by-step verification — is available here:

📎 [`AD_RBAC_Project_Report.pdf`](./AD_RBAC_Project_Report.pdf)

## 🔭 What's Next

This repo currently covers **RBAC (Role-Based Access Control)**. A follow-up phase will add:

- [ ] **Software Restriction Policy (SRP)** — blocking unauthorized application execution at the domain level, layered on top of the existing RBAC controls

## 🧠 Key Takeaway

> Group Policy enforcement in Active Directory is driven by **where an account lives in the OU structure** — not by the user's identity or group memberships alone. This lab proves that principle directly by moving a user between OUs and watching their effective permissions change instantly upon policy refresh.

---

<div align="center">

**Author:** Piyush Gupta · Computer Network Defense (CND) Project · September 2026

</div>
