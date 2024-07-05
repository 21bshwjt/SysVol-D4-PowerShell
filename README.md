# How to force authoritative and non-authoritative synchronization for DFSR-replicated sysvol replication
[Work in Progress]

Refer the MSFT KB : https://learn.microsoft.com/en-us/troubleshoot/windows-server/group-policy/force-authoritative-non-authoritative-synchronization

### 1. Set the DFS Replication service Startup Type to Manual and stop the service on all domain controllers in the domain. 
```powershell
$DCs = Get-ADGroupMember -Identity "Domain Controllers" | Select-Object -ExpandProperty Name 
# Change Service startup type manual & Stop the DRSR Service  

$DCs | Foreach-Object -Process { 
    #Action that will run in Parallel. Reference the current object via $PSItem and bring in outside variables with $USING:varname 
    Invoke-Command -ComputerName $PSItem { Set-Service -Name 'DFSR' -StartupType Manual -Verbose; Stop-Service -Name 'DFS Replication' -Force -Verbose 
    } 
} 
```

### 2. Verify DFSR Service Status from all Domain Controllers
```powershell
# Get the DFSR Service Status 
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

### 3. In the ADSIEDIT.MSC tool, modify the following DN and two attributes on the domain controller you want to make authoritative (preferably the PDC Emulator, which is usually the most up-to-date for sysvol replication contents)
```powershell
CN=SYSVOL Subscription,CN=Domain System Volume,CN=DFSR-LocalSettings,CN=<the server name>,OU=Domain Controllers,DC=<domain>

msDFSR-Enabled=FALSE
msDFSR-options=1
```

#### 3.1 Modify that using PowerShell
```powershell
# Change PDC on ADSIEDIT 
# Get the PDC Emulator for the domain 
$PDCNameFull = (Get-ADDomain).PDCEmulator 
 
# Split the full server name to get only the server name part 
$PDCName = $PDCNameFull -split '\.' | Select-Object -First 1 
 
$domain = (Get-ADDomain).DistinguishedName 
 
# Construct the DN (Distinguished Name) 
$dn = "CN=SYSVOL Subscription,CN=Domain System Volume,CN=DFSR-LocalSettings,CN=$PDCName,OU=Domain Controllers,$domain" 
 
# Set the attributes 
Set-ADObject -Identity $dn -Replace @{ 
    "msDFSR-Enabled" = $False 
    "msDFSR-options" = 1 
} -Verbose 
```



### Verify SysVol State
```powershell
<#
State values are:
0: Uninitialized
1: Initialized
2: Initial Sync
3: Auto Recovery
4: Normal
5: In Error
#>
$servers = Get-ADGroupMember -Identity "Domain Controllers" | Select-Object -ExpandProperty Name 

foreach ($server in $servers) {
    try {
        $result = Get-WmiObject -Namespace "root\microsoftdfs" -Class "dfsrreplicatedfolderinfo" -ComputerName $server -Filter "replicatedfoldername='SYSVOL share'" | 
        Select-Object @{Name = 'DomainController'; Expression = { $_.MemberName } }, ReplicationGroupName, ReplicatedFolderName, State
        if ($result) {
            $result # | Format-Table -AutoSize
        }
        else {
            Write-Host "No DFSR information found on $server for 'SYSVOL share'."
        }
    }
    catch {
        Write-Host "Error querying $server : $_"
    }
} 

```
