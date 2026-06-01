[README_1.md](https://github.com/user-attachments/files/28452473/README_1.md)
# metrotech-homelab
Personal IT home lab — Windows Server, Active Directory, OPNsense, PowerShell
# 🖥️ MetroTech Home Lab

> A personal IT home lab built on a Windows 11 laptop using VMware Workstation Pro — designed to develop hands-on skills in Windows Server, Active Directory, networking, cybersecurity, and cloud infrastructure.

![PowerShell](https://img.shields.io/badge/PowerShell-5.1%2B-blue?logo=powershell)
![VMware](https://img.shields.io/badge/VMware-Workstation%2017.6.3-orange?logo=vmware)
![Windows Server](https://img.shields.io/badge/Windows%20Server-2022-0078D6?logo=windows)
![Status](https://img.shields.io/badge/Status-Active%20Build-green)

---

## 📋 Overview

This lab is a self-contained virtualized environment running on a personal Windows 11 laptop. The goal is to build real enterprise IT skills without needing physical hardware — using VMware Workstation Pro to simulate a full network infrastructure including a Domain Controller, firewall, and attacker machine.

**Core learning objectives:**
- Windows Server 2022 & Active Directory (Domain Controller, DNS, DHCP, Group Policy)
- Networking (VLANs, routing, firewall configuration with pfSense)
- Cybersecurity & ethical hacking (Kali Linux, Metasploitable, attack/defense practice)
- Cloud (Azure / Entra ID hybrid identity, Intune MDM enrollment)

---

## 🖥️ Host Machine Specs

| Spec | Value |
|---|---|
| OS | Windows 11 Home — Build 26200 (24H2) |
| Processor | Intel Core i7 12th Gen — 10 cores, 12 logical |
| RAM | 16 GB |
| Storage | ~476 GB total, 382 GB free |
| Hypervisor | VMware Workstation Pro 17.6.3 |
| BIOS Mode | UEFI with Secure Boot enabled |

---

## 🌐 Network Architecture

```
Internet
    │
Home Router — OpenDNS
    │
Host Laptop (Wi-Fi)
    │
    ├── VMnet8 (NAT) ──────── Internet-connected VMs
    │        10.0.8.0/24
    │
    └── VMnet1 (Host-Only) ── Isolated lab network
             10.0.1.0/24
                  │
                  └── METROTECH-DC01 (Domain Controller)
                        Static IP: 10.0.1.10
                        DNS: Self
```

---

## 📦 VM Inventory

| VM Name | OS | Role | Status |
|---|---|---|---|
| METROTECH-DC01 | Windows Server 2022 Standard | Domain Controller | ✅ Complete |
| METROTECH-PC01 | Windows 11 | Domain-joined Client | ⏳ In Progress |
| METROTECH-FW01 | pfSense | Virtual Firewall | ⏳ Planned |
| METROTECH-KALI01 | Kali Linux | Penetration Testing | ⏳ Planned |
| METROTECH-SRV01 | Windows Server 2022 | Member Server | ⏳ Planned |

---

## 🏗️ Active Directory Structure

```
metrotech.lab
└── MetroTech (OU)
        ├── IT (OU)
        │     ├── [Admin User] → IT Admins group
        │     └── [HD Tech] → Help Desk group
        ├── Staff (OU)
        │     ├── [Staff User 1] → Staff Users group
        │     └── [Staff User 2] → Staff Users group
        ├── Computers (OU) — domain workstations
        └── Groups (OU)
              ├── IT Admins
              ├── Staff Users
              └── Help Desk
```

---

## ⚙️ Setup Process

### Phase 1 — Host Assessment via PowerShell

```powershell
# Confirm OS, processor, RAM
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, CsProcessors

# Check available disk space
Get-PSDrive C | Select-Object Name,
  @{N="Used(GB)";E={[math]::Round($_.Used/1GB,2)}},
  @{N="Free(GB)";E={[math]::Round($_.Free/1GB,2)}}

# Confirm VMware version
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* |
  Where-Object {$_.DisplayName -like "*VMware*"} |
  Select-Object DisplayName, DisplayVersion

# Review all network interfaces
Get-NetIPAddress |
  Where-Object {$_.AddressFamily -eq "IPv4"} |
  Select-Object InterfaceAlias, IPAddress, PrefixLength
```

---

### Phase 2 — VM Creation in VMware

1. New Virtual Machine → Typical
2. Attach ISO: Windows Server 2022 Evaluation
3. Select edition: **Standard Evaluation (Desktop Experience)**
4. Set computer name and Administrator password
5. Disk: 60 GB, single file
6. Uncheck "Power on after creation" to customize hardware first

**Hardware customization:**
- RAM: 4096 MB
- CPU: 1 processor / 2 cores
- Network: Host-Only (VMnet1)
- Remove floppy drive

---

### Phase 3 — VMX File Configuration via PowerShell

```powershell
$vmxPath = "C:\Users\[username]\Documents\Virtual Machines\[VM Name]\[VM Name].vmx"

# Ensure ISO connects at boot
Add-Content -Path $vmxPath -Value 'sata0:1.startConnected = "TRUE"'
Add-Content -Path $vmxPath -Value 'sata0:1.autodetect = "TRUE"'

# Add boot delay and set boot order
Add-Content -Path $vmxPath -Value 'bios.bootDelay = "5000"'
Add-Content -Path $vmxPath -Value 'bios.bootOrder = "cdrom,hdd,network"'

# After Windows installs — disconnect ISO
(Get-Content $vmxPath) -replace 'sata0:1.startConnected = "TRUE"',
  'sata0:1.startConnected = "FALSE"' | Set-Content $vmxPath
```

---

### Phase 4 — Initial Server Configuration

```powershell
# Rename the server
Rename-Computer -NewName "METROTECH-DC01" -Restart

# Set server description
Set-ItemProperty `
  -Path "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters" `
  -Name "srvcomment" `
  -Value "MetroTech Home Lab - Domain Controller 01"
```

---

### Phase 5 — Active Directory Setup

```powershell
# Install AD DS role
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Promote to Domain Controller
Install-ADDSForest `
  -DomainName "metrotech.lab" `
  -DomainNetbiosName "METROTECH" `
  -ForestMode "WinThreshold" `
  -DomainMode "WinThreshold" `
  -InstallDns:$true `
  -Force:$true

# Create OU structure
New-ADOrganizationalUnit -Name "MetroTech" -Path "DC=metrotech,DC=lab"
New-ADOrganizationalUnit -Name "IT" -Path "OU=MetroTech,DC=metrotech,DC=lab"
New-ADOrganizationalUnit -Name "Staff" -Path "OU=MetroTech,DC=metrotech,DC=lab"
New-ADOrganizationalUnit -Name "Computers" -Path "OU=MetroTech,DC=metrotech,DC=lab"
New-ADOrganizationalUnit -Name "Groups" -Path "OU=MetroTech,DC=metrotech,DC=lab"

# Create security groups
New-ADGroup -Name "IT Admins" -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=MetroTech,DC=metrotech,DC=lab"
New-ADGroup -Name "Staff Users" -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=MetroTech,DC=metrotech,DC=lab"
New-ADGroup -Name "Help Desk" -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=MetroTech,DC=metrotech,DC=lab"
```

---

### Phase 6 — Security Hardening

```powershell
# Password policy
Set-ADDefaultDomainPasswordPolicy -Identity "metrotech.lab" `
  -MinPasswordLength 8 `
  -PasswordHistoryCount 10 `
  -MaxPasswordAge "90.00:00:00" `
  -MinPasswordAge "1.00:00:00" `
  -ComplexityEnabled $true `
  -ReversibleEncryptionEnabled $false

# Fix lockout threshold
Set-ADDefaultDomainPasswordPolicy -Identity "metrotech.lab" `
  -LockoutThreshold 5 `
  -LockoutDuration "00:30:00" `
  -LockoutObservationWindow "00:30:00"

# Create and link Group Policy
New-GPO -Name "MetroTech Security Policy" -Comment "Screen lock and USB restrictions"
New-GPLink -Name "MetroTech Security Policy" -Target "OU=MetroTech,DC=metrotech,DC=lab"

# Login banner
Set-GPRegistryValue -Name "MetroTech Security Policy" -Key "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" -ValueName "LegalNoticeCaption" -Type String -Value "MetroTech Notice"
Set-GPRegistryValue -Name "MetroTech Security Policy" -Key "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" -ValueName "LegalNoticeText" -Type String -Value "This system is for authorized MetroTech users only. All activity is monitored and logged."

# Screen lock — 15 minutes
Set-GPRegistryValue -Name "MetroTech Security Policy" -Key "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" -ValueName "InactivityTimeoutSecs" -Type DWord -Value 900

# Disable USB storage
Set-GPRegistryValue -Name "MetroTech Security Policy" -Key "HKLM\SYSTEM\CurrentControlSet\Services\USBSTOR" -ValueName "Start" -Type DWord -Value 4

# Set static IP on DC
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress "10.0.1.10" -PrefixLength 24 -DefaultGateway "10.0.1.1"
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses "10.0.1.10"
```

---

## 🔧 Troubleshooting Log

### Issue 1 — EFI Network Timeout on Boot ✅ Resolved
**Cause:** VMX missing `startConnected = TRUE` — VM fell back to network boot.
**Fix:** Added `startConnected`, `autodetect`, and `bootOrder` via PowerShell `Add-Content`.
**Lesson:** Always verify `startConnected` in VMX when a VM won't boot from ISO.

---

### Issue 2 — Duplicate VMX Entry ✅ Resolved
**Cause:** `Add-Content` appends — running script twice created duplicate `bootDelay`.
**Fix:**
```powershell
$seen = $false
$cleaned = Get-Content $vmxPath | ForEach-Object {
    if ($_ -eq 'bios.bootDelay = "5000"') {
        if (-not $seen) { $seen = $true; $_ }
    } else { $_ }
}
$cleaned | Set-Content $vmxPath
```
**Lesson:** Use replacement logic with `Set-Content` instead of `Add-Content` for single-instance settings.

---

### Issue 3 — VM Process Running After GUI Close ✅ Resolved
**Cause:** `vmware-vmx.exe` stayed running in background.
**Fix:**
```powershell
Get-Process | Where-Object {$_.Name -like "*vmx*"} | Select-Object Name, Id
Stop-Process -Id [PID] -Force
```

---

### Issue 4 — Password Lockout Disabled ✅ Resolved
**Cause:** Default domain policy had `LockoutThreshold = 0` — unlimited attempts allowed.
**Fix:** Set threshold to 5 via `Set-ADDefaultDomainPasswordPolicy`.
**Lesson:** Always check lockout settings — default AD installs leave this disabled.

---

## 🗺️ Roadmap

### Immediate
- [ ] Purchase and activate Windows 11 Pro license
- [ ] Build METROTECH-PC01 — Windows 11 client VM
- [ ] Join PC01 to domain and test Group Policy

### Short Term
- [ ] Build METROTECH-KALI01 (Kali Linux)
- [ ] Build METROTECH-FW01 (pfSense firewall)
- [ ] Practice Group Policy — push desktop wallpaper, mapped drives
- [ ] Configure DHCP on DC for lab network

### Medium Term
- [ ] Connect lab AD to Azure Entra ID (hybrid identity)
- [ ] Practice Intune MDM enrollment
- [ ] Network+ VLAN lab using pfSense
- [ ] Kali vs Metasploitable attack/defense lab

---

## 📚 Resources

- [VMware Workstation Pro Documentation](https://docs.vmware.com/en/VMware-Workstation-Pro/index.html)
- [Windows Server 2022 Evaluation](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022)
- [Microsoft Learn — Active Directory](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview)
- [pfSense Documentation](https://docs.netgate.com/pfsense/en/latest/)
- [TryHackMe](https://tryhackme.com) — Hands-on cybersecurity labs
- [NetworkChuck YouTube](https://www.youtube.com/@NetworkChuck) — Networking and lab walkthroughs

---

## ⚠️ Disclaimer

This lab is for personal learning and skill development only. All IP addresses shown are generic placeholders. No sensitive configuration data is included in this repository.

---

*Built by MetroTech | Kansas City, MO*
