# Patch Management with WSUS and Group Policy

After a simulated ransomware attack that exploited unpatched systems, I built central patch management for a Windows domain. WSUS (Windows Server Update Services) downloads updates once from Microsoft, I approve what's needed, and Group Policy forces every client to install from my server on a schedule.

**Environment:** Windows Server 2022 domain controller + WSUS · Windows Server 2022 client (CLIENT01) · VMware · PowerShell, Group Policy Management, WSUS console, Windows Admin Center, Event Viewer
**Built as part of:** INFO33192 Windows Administration & Security, Sheridan College (Fall 2026)

## Result

The client registered with WSUS, scanned, and installed the October 2026 .NET cumulative update (KB5128928) **from my server, not from Microsoft**, confirmed by Event ID 19, update history, `WindowsUpdate.log` and the WSUS console. No client has stopped reporting.

## How it works

```
Microsoft Update ──► WSUS server (approve) ──► Group Policy points clients to WSUS ──► Client installs at 03:00 daily
```

## 1. Install and configure WSUS

- Added a dedicated 60 GB disk (D:) for update files, so patches can't fill the system drive.
- Installed WSUS with PowerShell, opened TCP 8530, synced only **Windows Server 2022** and only **Critical + Security** updates, which keeps storage and review time down.

```powershell
# Install the WSUS role and its management tools
Install-WindowsFeature -Name UpdateServices -IncludeManagementTools
# Tell WSUS to store update files on D:
& "C:\Program Files\Update Services\Tools\wsusutil.exe" postinstall CONTENT_DIR=D:\WSUSUpdates
# Allow clients to reach WSUS on its default port
New-NetFirewallRule -DisplayName "WSUS HTTP 8530" -Direction Inbound -Protocol TCP -LocalPort 8530 -Action Allow
```

![D: drive for updates](images/01-d-drive-for-updates.png)
![Sync succeeded](images/02-sync-succeeded.png)

## 2. Force clients to use WSUS with Group Policy

GPO `Lab5_WSUSPolicy`, linked to the clients' OU:
- **Configure Automatic Updates:** 4, auto download and install daily at 03:00.
- **Intranet update service:** my WSUS server on port 8530.
- **Do not connect to Windows Update Internet locations:** on, so clients can't bypass WSUS.
- **Detection frequency:** every hour, set with PowerShell.

**Problem I caught:** checking the GPO's real values showed `AUOptions = 3` (download and *notify*), which would leave patches waiting for a user to click. I fixed it to 4 and verified it on the client.

```powershell
# Read what the GPO actually sets, not what I think I clicked
Get-GPRegistryValue -Name "Lab5_WSUSPolicy" -Key "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate\AU"
# On the client: apply policy now, then list which GPOs applied
gpupdate /force
gpresult /scope computer /r
```

| Before | After |
|---|---|
| ![AUOptions 3](images/03-auoptions-3-before-fix.png) | ![GPO values after fix](images/04-gpo-values-after-fix.png) |

![Client gpresult](images/05-client-gpresult.png)
![Client registry AUOptions 4](images/06-client-registry-auoptions-4.png)

## 3. Approve the right updates with PowerShell

The client reported to WSUS, and the "needed" list held nearly every monthly update since 2024.

**Problem:** a broad approve-everything command approved **78 updates, about 24.5 GB**. Most were *superseded*: older cumulative updates fully replaced by newer ones.

**Fix:** declined the superseded ones, cancelled their downloads, ran cleanup, and approved only the two current October 2026 updates.

```powershell
$wsus = Get-WsusServer
# Decline every approved update that a newer one replaces
$wsus.GetUpdates() | Where-Object { $_.IsSuperseded -and $_.IsApproved } | ForEach-Object { $_.Decline() }
# Delete files WSUS no longer needs
Invoke-WsusServerCleanup -CleanupUnneededContentFiles
```

![Client reporting](images/07-client-reporting.png)

| The problem | The fix |
|---|---|
| ![78 approved](images/08-78-updates-approved.png) | ![Superseded declined](images/09-superseded-declined.png) |

![Console shows 2 approved](images/10-console-2-approved.png)

**Second problem:** WSUS reported the October OS update as *Not applicable* to this client's older build, while the .NET update was *Needed*. WSUS only offers what a client's own scan finds applicable, so I removed the OS approval and deployed the .NET update.

![Applicability per update](images/11-applicability-per-kb.png)

## 4. Prove it worked

```powershell
# Find the WSUS server address in the client's update log
Get-WindowsUpdateLog
Select-String -Path "$env:USERPROFILE\Desktop\WindowsUpdate.log" -Pattern "8530"
# Install-success events (ID 19) are in the System log on Server 2016+
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Microsoft-Windows-WindowsUpdateClient'; Id=19} -MaxEvents 5
# Any client that hasn't reported in 7 days? (none = healthy)
Get-WsusComputer -ToLastReportedStatusTime (Get-Date).AddDays(-7)
```

![Update history](images/12-client-update-history.png)
![WindowsUpdate.log shows WSUS](images/13-windowsupdate-log-8530.png)
![Event 19](images/14-event-19-installed.png)
![No stale clients](images/15-no-stale-clients.png)

## Event IDs I used

| Event ID | Where | Meaning |
|---|---|---|
| 1502 / 1500 | System – GroupPolicy | Computer policy applied / no changes |
| 26 | WindowsUpdateClient/Operational | Client scan found updates |
| 41 | WindowsUpdateClient/Operational | Update downloaded |
| 19 | System – WindowsUpdateClient | Installation successful |
| 20 | System – WindowsUpdateClient | Installation failed |

## What I learned

- **Verify on the target, not in the editor.** The GPO looked right but was set to notify-only.
- **Approve less, not more.** Superseded updates waste storage and bandwidth and make review harder.
- **Automation is how patching scales.** The same PowerShell rules (approve only current Security updates, check for stale clients) can run on a schedule across thousands of machines.
- **In production** I'd approve to a pilot group first, then roll out to everyone.

More problems from this lab are in my [IT Troubleshooting Log](https://github.com/ikassem-cyber/it-troubleshooting-log/blob/main/cases/04-wsus-patching.md).
