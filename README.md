<div align="center">

<img width="80" src="https://github.com/21bshwjt/SysVol-D4-PowerShell/blob/bf9cb4a2ecc57a5f5b4b0a411ddcc0ab53f3e607/Screenshots/dfsr.png?raw=true">

# Force DFSR SysVol Replication via PowerShell

*Automate the D4/D2 SysVol replication recovery process across all Domain Controllers — no manual ADSIEDIT required.*

[![PowerShell](https://img.shields.io/badge/PowerShell-5.1%2B-5391FE?style=for-the-badge&logo=powershell&logoColor=white)](https://learn.microsoft.com/en-us/powershell/)
[![Active Directory](https://img.shields.io/badge/Active%20Directory-Required-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview)
[![Windows Server](https://img.shields.io/badge/Windows%20Server-2016%2B-0078D4?style=for-the-badge&logo=windows&logoColor=white)](https://www.microsoft.com/en-us/windows-server)
[![License](https://img.shields.io/badge/License-MIT-brightgreen?style=for-the-badge)](LICENSE)

---

📖 **Official Microsoft KB** → [Force Authoritative & Non-Authoritative Synchronization for DFSR-Replicated Sysvol](https://learn.microsoft.com/en-us/troubleshoot/windows-server/group-policy/force-authoritative-non-authoritative-synchronization)

</div>

---

## 📋 Table of Contents

| # | Step | Description |
|---|---|---|
| 1 | [Stop DFSR Service](#-step-1--stop-dfsr-service-on-all-dcs) | Set startup to Manual & stop on all DCs |
| 2 | [Verify Service Status](#-step-2--verify-dfsr-service-status) | Confirm service is stopped across all DCs |
| 3 | [Manual ADSIEDIT Reference](#-step-3--manual-adsiedit-reference) | Reference for manual attribute changes |
| 4 | [Set PDC as Authoritative](#-step-4--set-pdc-as-authoritative-automated) | Automate ADSIEDIT changes on PDC |
| 5 | [Set All Other DCs](#-step-5--set-msdfsrenabled--false-on-all-other-dcs) | Set `msDFSR-Enabled=FALSE` on non-PDC DCs |
| 6 | [Force AD Replication](#-step-6--force-ad-replication) | Sync changes across the domain |
| 7 | [Start DFSR on PDC](#-step-7--start-dfsr-on-pdc) | Bring up replication on the authoritative DC |
| 8 | [Event ID 4114](#-step-8--event-id-4114) | Verify sysvol is no longer replicating |
| 9 | [Re-enable PDC](#-step-9--set-msdfsrenabled--true-on-pdc) | Set `msDFSR-Enabled=TRUE` on PDC |
| 10 | [Force AD Replication Again](#-step-10--force-ad-replication-again) | Push updated attributes across domain |
| 11 | [DFSRDIAG on PDC](#-step-11--run-dfsrdiag-pollad-on-pdc) | Trigger DFSR poll on authoritative DC |
| 12 | [Event ID 4602](#-step-12--event-id-4602) | Confirm D4 initialization complete |
| 13 | [Start DFSR on Other DCs](#-step-13--start-dfsr-on-all-non-authoritative-dcs) | Bring up replication on all remaining DCs |
| 14 | [Re-enable All Other DCs](#-step-14--set-msdfsrenabled--true-on-all-other-dcs) | Set `msDFSR-Enabled=TRUE` on non-PDC DCs |
| 15 | [DFSRDIAG on Non-Auth DCs](#-step-15--run-dfsrdiag-pollad-on-all-non-auth-dcs) | Trigger DFSR poll on all non-authoritative DCs |
| 16 | [Restore Service to Automatic](#-step-16--restore-dfsr-startup-type-to-automatic) | Return DFSR to automatic startup on all DCs |
| 17 | [Verify Final Status](#-step-17--verify-final-dfsr-service-status) | Confirm all DCs are healthy |
| 18 | [SysVol Health Check](#-step-18--sysvol-health-check-post-validation) | Validate SysVol state = 4 (Normal) on all DCs |
| 19 | [Verify ADSIEDIT Attributes](#-step-19--verify-adsiedit-attribute-values-optional) | Confirm `msDFSR-options` reset to 0 |

---

## ⚠️ Use Cases

```diff
- 1. Missing SysVol / Netlogon folders on one or more Domain Controllers
- 2. GPO inconsistencies across Domain Controllers in the domain
```

---

## ✅ Prerequisites

> Run all scripts **in sequence** from the **PDC Emulator** with an elevated PowerShell session.

| Requirement | Detail |
|---|---|
| 🖥️ Run Location | PDC Emulator |
| 🔑 Permissions | Domain Admins |
| 📦 Module | Active Directory PowerShell Module |
| 📂 Setup | Copy the `Scripts` folder to the PDC |
| 🔢 Script Order | Run in sequence — scripts 3, 8 & 12 are intentionally absent (manual/event steps) |
| ✔️ Post-Validation | Scripts 18 and 19 are for post-validation only |

---

## 🔵 Step 1 — Stop DFSR Service on All DCs

> Set the DFS Replication service startup type to **Manual** and stop it on **all domain controllers**.

```powershell
$DCs = Get-ADGroupMember -Identity "Domain Controllers" | Select-Object -ExpandProperty Name

$DCs | ForEach-Object -Process {
    try {
        Invoke-Command -ComputerName $PSItem -ScriptBlock {
            Set-Service -Name 'DFSR' -StartupType Manual -Verbose
            Stop-Service -Name 'DFS Replication' -Force -Verbose
        } -ErrorAction Stop
    } catch {
        Write-Error "Failed to modify DFSR service on $PSItem | Error: $_"
    }
}
```

---

## 🔵 Step 2 — Verify DFSR Service Status

> Confirm the service is **stopped** and set to **Manual** on all DCs before proceeding.

```powershell
$DCs = Get-ADGroupMember -Identity "Domain Controllers" | Select-Object -ExpandProperty Name

$GetoBj = Foreach ($DC in $DCs) {
    Invoke-Command -ComputerName $DC {
        [PSCustomObject]@{
            DomainController = ($env:COMPUTERNAME).ToUpper()
            ServiceName      = (Get-Service -Name DFSR).Name
            Status           = (Get-Service -Name DFSR).Status
            StartType        = (Get-Service -Name DFSR).StartType
        }
    }
}
$GetoBj | Select-Object -Property DomainController, ServiceName, Status, StartType
```

---

## 🟡 Step 3 — Manual ADSIEDIT Reference

> ℹ️ This step is handled **automatically in Step 4**. Shown here for reference only.

In `ADSIEDIT.MSC`, modify the following DN on the **PDC Emulator**:

```
CN=SYSVOL Subscription,CN=Domain System Volume,CN=DFSR-LocalSettings,CN=<ServerName>,OU=Domain Controllers,DC=<domain>
```

| Attribute | Value |
|---|---|
| `msDFSR-Enabled` | `FALSE` |
| `msDFSR-options` | `1` |

---

## 🔵 Step 4 — Set PDC as Authoritative *(Automated)*

> Automatically apply the ADSIEDIT changes to the **PDC Emulator** via PowerShell.

```powershell
$PDCNameFull = (Get-ADDomain).PDCEmulator
$PDCName     = $PDCNameFull -split '\.' | Select-Object -First 1
$domain      = (Get-ADDomain).DistinguishedName
$dn          = "CN=SYSVOL Subscription,CN=Domain System Volume,CN=DFSR-LocalSettings,CN=$PDCName,OU=Domain Controllers,$domain"

Set-ADObject -Identity $dn -Replace @{
    "msDFSR-Enabled" = $False
    "msDFSR-options" = 1
} -Verbose
```

---

## 🔵 Step 5 — Set `msDFSR-Enabled = FALSE` on All Other DCs

> Disable DFSR replication on **all non-PDC domain controllers**.

```powershell
$domain = (Get-ADDomain).DistinguishedName
$DCs    = Get-ADGroupMember -Identity "Domain Controllers" | Select-Object -ExpandProperty Name

foreach ($DC in $DCs) {
    $dn = "CN=SYSVOL Subscription,CN=Domain System Volume,CN=DFSR-LocalSettings,CN=$DC,OU=Domain Controllers,$domain"
    Set-ADObject -Identity $dn -Replace @{
        "msDFSR-Enabled" = $False
    } -Verbose
}
```

---

## 🔵 Step 6 — Force AD Replication

> Push all attribute changes across the entire domain and validate replication success.

```powershell
repadmin /syncall /A /e /P /d /q
```

---

## 🔵 Step 7 — Start DFSR on PDC

> Start the DFS Replication service **only on the PDC Emulator** at this stage.

```powershell
$PDCNameFull = (Get-ADDomain).PDCEmulator
$PDCName     = $PDCNameFull -split '\.' | Select-Object -First 1

Invoke-Command -ComputerName $PDCName {
    Start-Service -Name 'DFS Replication' -Verbose
}
```

---

## 🟡 Step 8 — Event ID 4114

> 📋 **No script required.** Open **Event Viewer → DFS Replication** on the PDC.
>
> ✅ You should see **Event ID 4114** — confirming SysVol replication is no longer being replicated (expected at this stage).

---

## 🔵 Step 9 — Set `msDFSR-Enabled = TRUE` on PDC

> Re-enable DFSR on the PDC Emulator to trigger the **authoritative (D4) restore**.

```powershell
$PDCNameFull = (Get-ADDomain).PDCEmulator
$PDCName     = $PDCNameFull -split '\.' | Select-Object -First 1
$domain      = (Get-ADDomain).DistinguishedName
$dn          = "CN=SYSVOL Subscription,CN=Domain System Volume,CN=DFSR-LocalSettings,CN=$PDCName,OU=Domain Controllers,$domain"

Set-ADObject -Identity $dn -Replace @{
    "msDFSR-Enabled" = $True
} -Verbose
```

---

## 🔵 Step 10 — Force AD Replication Again

> Sync the updated attributes across the domain once more.

```powershell
repadmin /syncall /A /e /P /d /q
```

---

## 🔵 Step 11 — Run `DFSRDIAG POLLAD` on PDC

> Force the PDC to poll Active Directory and apply the new DFSR configuration.

```powershell
DFSRDIAG POLLAD
```

---

## 🟡 Step 12 — Event ID 4602

> 📋 **No script required.** Open **Event Viewer → DFS Replication** on the PDC.
>
> ✅ You should see **Event ID 4602** — confirming SysVol replication has been **initialized**. The PDC has completed the **D4 authoritative restore**.

---

## 🔵 Step 13 — Start DFSR on All Non-Authoritative DCs

> Start the DFS Replication service on **all remaining domain controllers**.

```powershell
$DCs = Get-ADGroupMember -Identity "Domain Controllers" | Select-Object -ExpandProperty Name

$DCs | ForEach-Object -Process {
    Invoke-Command -ComputerName $PSItem {
        Start-Service -Name 'DFS Replication' -Verbose
    }
}
```

> 📋 After starting, check **Event Viewer → DFS Replication** on each DC for **Event ID 4114** (expected).

---

## 🔵 Step 14 — Set `msDFSR-Enabled = TRUE` on All Other DCs

> Re-enable DFSR replication on all **non-authoritative domain controllers**.

```powershell
$domain = (Get-ADDomain).DistinguishedName
$DCs    = Get-ADGroupMember -Identity "Domain Controllers" | Select-Object -ExpandProperty Name

foreach ($DC in $DCs) {
    $dn = "CN=SYSVOL Subscription,CN=Domain System Volume,CN=DFSR-LocalSettings,CN=$DC,OU=Domain Controllers,$domain"
    Set-ADObject -Identity $dn -Replace @{
        "msDFSR-Enabled" = $True
    } -Verbose
}
```

---

## 🔵 Step 15 — Run `DFSRDIAG POLLAD` on All Non-Auth DCs

> Force all non-authoritative DCs to poll AD and pull SysVol content from the PDC.

```powershell
$servers     = Get-ADGroupMember -Identity "Domain Controllers" | Select-Object -ExpandProperty Name
$PDCNameFull = (Get-ADDomain).PDCEmulator
$PDCName     = $PDCNameFull -split '\.' | Select-Object -First 1

# Exclude PDC from the list
$servers = $servers | Where-Object { $_ -ne $PDCName }

$servers | ForEach-Object -Process {
    Invoke-Command -ComputerName $PSItem { DFSRDIAG POLLAD -Verbose }
}
```

---

## 🔵 Step 16 — Restore DFSR Startup Type to Automatic

> Return the DFSR service to **Automatic** startup on all domain controllers.

```powershell
$DCs = Get-ADGroupMember -Identity "Domain Controllers" | Select-Object -ExpandProperty Name

$DCs | ForEach-Object -Process {
    Invoke-Command -ComputerName $PSItem {
        Set-Service -Name 'DFSR' -StartupType Automatic -Verbose
    }
}
```

---

## 🔵 Step 17 — Verify Final DFSR Service Status

> Confirm DFSR is **Running** and set to **Automatic** on all DCs.

```powershell
$DCs = Get-ADGroupMember -Identity "Domain Controllers" | Select-Object -ExpandProperty Name

$GetoBj = foreach ($DC in $DCs) {
    try {
        Invoke-Command -ComputerName $DC -ScriptBlock {
            [PSCustomObject]@{
                DomainController = $env:COMPUTERNAME.ToUpper()
                ServiceName      = (Get-Service -Name DFSR -ErrorAction Stop).Name
                Status           = (Get-Service -Name DFSR -ErrorAction Stop).Status
                StartType        = (Get-Service -Name DFSR -ErrorAction Stop).StartType
            }
        }
    } catch {
        [PSCustomObject]@{
            DomainController = $DC.ToUpper()
            ServiceName      = "DFSR"
            Status           = "Error: $($Error[0].Exception.Message)"
            StartType        = "Unknown"
        }
    }
}
$GetoBj | Select-Object -Property DomainController, ServiceName, Status, StartType
```

---

## 🟢 Step 18 — SysVol Health Check *(Post-Validation)*

```diff
+ Expected "State" value on all DCs is 4 (Normal) after replication completes.
```

| State Value | Meaning |
|---|---|
| `0` | Uninitialized |
| `1` | Initialized |
| `2` | Initial Sync |
| `3` | Auto Recovery |
| ✅ `4` | **Normal** ← Expected |
| `5` | In Error |

```powershell
$servers = Get-ADGroupMember -Identity "Domain Controllers" | Select-Object -ExpandProperty Name

foreach ($server in $servers) {
    try {
        $result = Get-WmiObject -Namespace "root\microsoftdfs" -Class "dfsrreplicatedfolderinfo" `
            -ComputerName $server -Filter "replicatedfoldername='SYSVOL share'" |
            Select-Object @{Name = 'DomainController'; Expression = { $_.MemberName } },
                          ReplicationGroupName, ReplicatedFolderName, State
        if ($result) {
            $result
        } else {
            Write-Warning "No DFSR info found on $server for 'SYSVOL share'."
        }
    } catch {
        Write-Warning "Error querying $server : $_"
    }
}
```

---

## 🟢 Step 19 — Verify ADSIEDIT Attribute Values *(Optional)*

```diff
+ msDFSR-options will revert automatically from "1" back to "0" on the PDC after some time.
```

```powershell
$domain = (Get-ADDomain).DistinguishedName
$DCs    = Get-ADGroupMember -Identity "Domain Controllers" | Select-Object -ExpandProperty Name

$Objs = Foreach ($DC in $DCs) {
    Get-ADObject -Filter { Name -eq "SYSVOL Subscription" } `
        -SearchBase "CN=Domain System Volume,CN=DFSR-LocalSettings,CN=$DC,OU=Domain Controllers,$domain" `
        -Properties DistinguishedName, msDFSR-Enabled, msDFSR-options |
        Select-Object DistinguishedName, msDFSR-Enabled, msDFSR-options
}

foreach ($Obj in $Objs) {
    $msDFSR_options = $Obj.'msDFSR-options'
    if ([string]::IsNullOrWhiteSpace($msDFSR_options)) { $msDFSR_options = "<not set>" }

    [PSCustomObject]@{
        DomainController = ($Obj.DistinguishedName -split ",")[3].Substring(3)
        "msDFSR-Enabled" = $Obj.'msDFSR-Enabled'
        "msDFSR-options" = $msDFSR_options
    }
}
```

---

<div align="center">

## 🔁 Process Flow at a Glance

```
Stop DFSR (All DCs)
       ↓
Set PDC → msDFSR-Enabled=FALSE, msDFSR-options=1
       ↓
Set All Other DCs → msDFSR-Enabled=FALSE
       ↓
Force AD Replication
       ↓
Start DFSR on PDC → [Event ID 4114]
       ↓
Set PDC → msDFSR-Enabled=TRUE
       ↓
Force AD Replication
       ↓
DFSRDIAG POLLAD on PDC → [Event ID 4602 — D4 Complete ✅]
       ↓
Start DFSR on All Other DCs → [Event ID 4114 on each]
       ↓
Set All Other DCs → msDFSR-Enabled=TRUE
       ↓
DFSRDIAG POLLAD on All Non-Auth DCs
       ↓
Restore DFSR to Automatic (All DCs)
       ↓
Post-Validation: SysVol State = 4 ✅
```

---

## 👤 Author

**Biswajit Biswas** *(a.k.a. bshwjt)*

[![LinkedIn](https://img.shields.io/badge/LinkedIn-bshwjt-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/bshwjt/)
[![Email](https://img.shields.io/badge/Email-bshwjt%40gmail.com-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:bshwjt@gmail.com)
[![GitHub](https://img.shields.io/badge/GitHub-21bshwjt-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/21bshwjt)

---

*📌 All scripts are provided as-is. Always test in a non-production environment first.*

[![Maintained](https://img.shields.io/badge/Maintained-Yes-brightgreen?style=flat-square)](https://github.com/21bshwjt)
[![PowerShell](https://img.shields.io/badge/Built%20with-PowerShell-5391FE?style=flat-square&logo=powershell)](https://learn.microsoft.com/en-us/powershell/)

</div>
