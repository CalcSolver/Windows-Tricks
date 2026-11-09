# ⚡ Awesome Windows Power Tools

> Surprising, lesser-known, "wait, Windows can do *that*?" commands for power users, developers and sysadmins.
> 100 core tricks + 80 bonus entries (one-liners, hidden tools, automation scripts, fun, IT-pro).

![Platform](https://img.shields.io/badge/platform-Windows%2010%20%7C%2011-blue) ![PowerShell](https://img.shields.io/badge/PowerShell-5.1%20%7C%207%2B-5391FE) ![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen)

## ⚠️ Read first

- Run **elevated** (Admin) where noted. Many diagnostics return partial data without it.
- Test on a VM or [Windows Sandbox](#73-windows-sandbox-disposable-desktop-with-a-wsb-config) before running anything 🟡/🔴 on a machine you care about.
- Commands are shown for **PowerShell** unless the block says `cmd`. In PowerShell, call `curl.exe`, not `curl` (an alias for `Invoke-WebRequest` in 5.1).
- Items marked **(PS7)** need PowerShell 7+ (`winget install Microsoft.PowerShell`).
- Some features need a recent Windows 11 build; this is noted per item.

### Danger levels

| Level | Meaning |
|---|---|
| 🟢 **Safe** | Read-only or trivially reversible |
| 🟡 **Moderate** | Changes settings, deletes data, reboots, or uses lots of resources. Understand it first |
| 🔴 **Advanced** | Can break the OS, lose data, or weaken security if misused. Have a backup/restore point |

## 📚 Table of contents

**Core 100**
1. [System info & visualization](#1-system-info--visualization) (1–10)
2. [Storage hogs](#2-storage-hogs) (11–20)
3. [Boot & performance](#3-boot--performance) (21–28)
4. [Reports](#4-reports) (29–36)
5. [Networking](#5-networking) (37–48)
6. [Repair & maintenance (DISM / SFC / CHKDSK)](#6-repair--maintenance) (49–56)
7. [PowerCfg](#7-powercfg) (57–64)
8. [Sysinternals](#8-sysinternals) (65–72)
9. [Sandbox & Hyper-V](#9-windows-sandbox--hyper-v) (73–78)
10. [Task Scheduler & Event Viewer](#10-task-scheduler--event-viewer) (79–86)
11. [Registry tweaks](#11-registry-tweaks) (87–93)
12. [Winget, Terminal & productivity](#12-winget-terminal--productivity) (94–100)

**Bonus**
- [20 PowerShell one-liners](#-20-powershell-one-liners)
- [20 hidden Windows tools](#-20-hidden-windows-tools)
- [20 GitHub-worthy automation scripts](#-20-automation-scripts)
- [10 fun (safe) commands](#-10-fun-safe-commands)
- [10 commands IT professionals live by](#-10-commands-it-professionals-live-by)

---

# 1. System info & visualization

#### 1. Full System Information Report to a Text File
```powershell
msinfo32 /report "$HOME\Desktop\msinfo.txt"
```
**What:** Dumps the whole System Information database (hardware, drivers, services, environment, startup programs) to a file without opening the GUI.
**Why:** A single, attachable snapshot for support tickets or before/after comparisons.
**Danger:** 🟢 Safe

#### 2. Interactive, Filterable Services Grid
```powershell
Get-CimInstance Win32_Service | Select Name,State,StartMode,StartName,PathName | Out-GridView -Title Services -PassThru
```
**What:** Opens a sortable, searchable GUI grid; select rows and press OK to pass them down the pipeline.
**Why:** Instantly spot services running as odd accounts or from unusual paths, and use `-PassThru` to feed selections into `Stop-Service` etc. *(Out-GridView is Windows PowerShell 5.1 or PS7 with `Microsoft.PowerShell.GraphicalTools`.)*
**Danger:** 🟢 Safe

#### 3. Live Top CPU Consumers From Performance Counters
```powershell
Get-Counter '\Process(*)\% Processor Time' | Select -Expand CounterSamples |
  ? InstanceName -notin '_total','idle' | sort CookedValue -Desc |
  select -First 5 InstanceName,@{n='CPU%';e={[math]::Round($_.CookedValue,1)}}
```
**What:** Reads raw perf counters, the same source as Task Manager and perfmon, and ranks processes by instantaneous CPU.
**Why:** Scriptable "top" for Windows that also works remotely with `-ComputerName`. Values are per-core percentages, so they can exceed 100.
**Danger:** 🟢 Safe

#### 4. RAM Module Inventory (Speed, Slot, Part Number)
```powershell
Get-CimInstance Win32_PhysicalMemory | Select BankLabel,@{n='GB';e={$_.Capacity/1GB}},Speed,ConfiguredClockSpeed,Manufacturer,PartNumber
```
**What:** Lists every DIMM with its rated and configured speed.
**Why:** Verify XMP/EXPO is actually on (Speed ≠ ConfiguredClockSpeed), and buy matching sticks without opening the case.
**Danger:** 🟢 Safe

#### 5. SSD/HDD Health & Wear Counters
```powershell
Get-PhysicalDisk | Get-StorageReliabilityCounter | Select DeviceId,Temperature,Wear,PowerOnHours,ReadErrorsTotal,WriteErrorsTotal
```
**What:** Pulls SMART-style reliability data via the Storage module (run as Admin).
**Why:** Catch a dying drive early with no third-party tool. Not every controller exposes every field.
**Danger:** 🟢 Safe

#### 6. Memory Usage Grouped by Process Name
```powershell
Get-Process | Group Name | % { [pscustomobject]@{Name=$_.Name;Instances=$_.Count;MB=[math]::Round(($_.Group|Measure WorkingSet64 -Sum).Sum/1MB)} } | sort MB -Desc | select -First 10
```
**What:** Collapses 40 Chrome/Edge/Teams processes into one row each with the total RAM.
**Why:** Shows the *real* memory hogs. This is the advanced use of `Get-Process`.
**Danger:** 🟢 Safe

#### 7. ACPI Thermal Zone Temperature
```powershell
Get-CimInstance -Namespace root/wmi MSAcpi_ThermalZoneTemperature | Select InstanceName,@{n='°C';e={[math]::Round($_.CurrentTemperature/10-273.15,1)}}
```
**What:** Reads the firmware thermal zone through WMI (Admin).
**Why:** Quick heat check with zero software installed. Many desktops don't expose it, and it's not the CPU die sensor.
**Danger:** 🟢 Safe

#### 8. Resource Monitor From the Run Box
```cmd
perfmon /res
```
**What:** Opens Resource Monitor directly.
**Why:** Per-process disk/network I/O and **"Associated Handles"** search. Type a filename to see which process locks it.
**Danger:** 🟢 Safe

#### 9. Reliability Stability Index History
```powershell
Get-CimInstance Win32_ReliabilityStabilityMetrics | Select -First 14 StartMeasurementDate,SystemStabilityIndex
```
**What:** The numeric history behind Reliability Monitor (`perfmon /rel`): a 1–10 score per day.
**Why:** Pinpoints the day a driver or update started causing crashes.
**Danger:** 🟢 Safe

#### 10. Built-in Hardware Benchmark (WinSAT)
```powershell
winsat formal
Get-CimInstance Win32_WinSAT | Select CPUScore,MemoryScore,GraphicsScore,D3DScore,DiskScore
```
**What:** Runs the hidden Windows System Assessment Tool and stores scores in WMI (Admin).
**Why:** The "Windows Experience Index" never died, it just lost its UI. Good for quick before/after hardware comparisons.
**Danger:** 🟢 Safe *(saturates CPU/disk for a few minutes)*

---

# 2. Storage hogs

#### 11. Top 20 Largest Files on a Drive
```powershell
gci C:\ -Recurse -File -Force -EA SilentlyContinue | sort Length -Desc | select -First 20 FullName,@{n='GB';e={[math]::Round($_.Length/1GB,2)}}
```
**What:** Walks the drive and ranks files by size.
**Why:** Finds forgotten VM disks, `.dmp` files and installers. Slow on big drives; scope it to `C:\Users`.
**Danger:** 🟢 Safe

#### 12. Folder Size Leaderboard
```powershell
gci C:\Users -Directory | % { [pscustomobject]@{Folder=$_.Name;GB=[math]::Round((gci $_.FullName -Recurse -File -Force -EA 0 | Measure Length -Sum).Sum/1GB,2)} } | sort GB -Desc
```
**What:** Sums each child folder's size.
**Why:** Poor-man's WinDirStat in one line. Point it at `AppData\Local` to find cache monsters.
**Danger:** 🟢 Safe

#### 13. Find Duplicate Files by Hash
```powershell
gci D:\Photos -Recurse -File | Get-FileHash -Algorithm SHA256 | Group Hash | ? Count -gt 1 | % { $_.Group.Path; '' }
```
**What:** Groups files by SHA-256 and prints every set of identical files.
**Why:** Reclaim space without trusting file names or dates. Hashing is I/O heavy on large trees.
**Danger:** 🟢 Safe

#### 14. Large Files Nobody Has Touched in a Year
```powershell
gci D:\ -Recurse -File -EA 0 | ? { $_.LastWriteTime -lt (Get-Date).AddYears(-1) -and $_.Length -gt 100MB } | sort Length -Desc | select -First 25 FullName,@{n='MB';e={[int]($_.Length/1MB)}}
```
**What:** Big + old = archive candidates.
**Why:** Uses `LastWriteTime`, which is reliable (unlike `LastAccessTime`; check `fsutil behavior query disablelastaccess`).
**Danger:** 🟢 Safe

#### 15. Analyze the WinSxS Component Store
```cmd
Dism /Online /Cleanup-Image /AnalyzeComponentStore
```
**What:** Reports the *real* size of WinSxS (File Explorer overstates it because of hardlinks) and whether cleanup is recommended.
**Why:** Answers "is WinSxS huge?" definitively before you touch anything.
**Danger:** 🟢 Safe

#### 16. Reclaim Space From the Component Store
```cmd
Dism /Online /Cleanup-Image /StartComponentCleanup /ResetBase
```
**What:** Removes superseded component versions and **makes installed updates permanent**.
**Why:** Can recover GBs on long-lived installs. After `/ResetBase` you cannot uninstall existing updates.
**Danger:** 🔴 Advanced

#### 17. Scripted Disk Cleanup With Saved Selections
```cmd
cleanmgr /sageset:65
cleanmgr /sagerun:65
```
**What:** `sageset` stores which categories to clean under profile 65; `sagerun` replays it silently.
**Why:** Repeatable cleanup via Task Scheduler. Includes categories hidden from the normal UI (old Windows installs, Delivery Optimization, driver packages).
**Danger:** 🟡 Moderate

#### 18. Check and Cap Shadow Copy (Restore Point) Storage
```cmd
vssadmin list shadowstorage
vssadmin resize shadowstorage /for=C: /on=C: /maxsize=10GB
```
**What:** Shows how much space VSS snapshots use and caps it.
**Why:** System Restore and File History snapshots can silently eat tens of GB.
**Danger:** 🟡 Moderate *(old restore points are dropped to honor the cap)*

#### 19. Shrink hiberfil.sys
```cmd
powercfg /h /type reduced
```
**What:** Reduces the hibernation file (about 20% of RAM) and keeps only Fast Startup support. Full hibernate becomes unavailable.
**Why:** Reclaims 6–25 GB on high-RAM machines. `powercfg /h /type full` reverses it.
**Danger:** 🟡 Moderate

#### 20. CompactOS: Compress the Windows Install
```cmd
compact /compactOS:query
compact /compactOS:always
```
**What:** Compresses OS binaries with a fast algorithm.
**Why:** Frees 1–3 GB on small SSDs and tablets with negligible CPU cost. `/compactOS:never` reverses it.
**Danger:** 🟡 Moderate

---

# 3. Boot & performance

#### 21. Last 10 Boot Times in Seconds
```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Diagnostics-Performance/Operational';Id=100} -MaxEvents 10 |
 % { $d=([xml]$_.ToXml()).Event.EventData.Data; [pscustomobject]@{Time=$_.TimeCreated;BootSec=[math]::Round([int](($d|? Name -eq 'BootTime').'#text')/1000,1)} }
```
**What:** Reads Windows' own boot-performance telemetry (Event ID 100).
**Why:** Charts boot time regressions across updates and driver changes.
**Danger:** 🟢 Safe

#### 22. Apps That Slowed Your Boot
```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Diagnostics-Performance/Operational';Id=101} -MaxEvents 15 |
 % { $d=([xml]$_.ToXml()).Event.EventData.Data; [pscustomobject]@{Time=$_.TimeCreated;App=($d|? Name -eq 'Name').'#text';TotalMs=($d|? Name -eq 'TotalTime').'#text'} }
```
**What:** Event 101 records apps that degraded boot, with how long they took. (IDs 102/103/106 cover drivers/services/devices.)
**Why:** Data-driven answer to "what should I disable?". Field names can vary by build; inspect `$_.ToXml()` if empty.
**Danger:** 🟢 Safe

#### 23. Record a Full Boot Trace With WPR
```cmd
wpr -boottrace -addboot GeneralProfile -filemode
:: reboot, then:
wpr -boottrace -stopboot C:\boot.etl
```
**What:** Windows Performance Recorder captures kernel-level boot activity to an ETL for Windows Performance Analyzer (`winget install Microsoft.WindowsPerformanceToolkit`).
**Why:** The gold-standard method used by Microsoft engineers. Shows exactly which driver or service stalls boot.
**Danger:** 🔴 Advanced

#### 24. Everything That Starts With Windows, in One List
```powershell
Get-CimInstance Win32_StartupCommand | Select Name,Command,Location,User | Sort Location | Format-Table -Auto
```
**What:** Run keys, startup folders, per-user and machine-wide.
**Why:** Quick audit with no GUI. For a fuller view (tasks, services, drivers) see [Autoruns](#70-autoruns-from-the-command-line).
**Danger:** 🟢 Safe

#### 25. 60-Second System Diagnostics Report
```cmd
perfmon /report
```
**What:** Collects data for about 60 seconds, then produces an HTML report with warnings and a resource overview.
**Why:** Pass/fail checks for CPU, disk, memory, network and services. Good first triage on a "slow PC".
**Danger:** 🟢 Safe

#### 26. Find What Defender Scans Are Slowing Down
```powershell
New-MpPerformanceRecording -RecordTo C:\defender.etl      # press Enter to stop
Get-MpPerformanceReport -Path C:\defender.etl -TopFiles 10 -TopScans 10 -TopProcesses 10
```
**What:** Records Defender activity and ranks the slowest files, scans and processes.
**Why:** Gives hard data for *targeted* exclusions (compilers, `node_modules`) instead of guesses. Requires Admin.
**Danger:** 🟢 Safe *(exclusions you add afterward are a trade-off)*

#### 27. Reboot Straight Into UEFI Firmware or Recovery
```cmd
shutdown /r /fw /t 0
shutdown /r /o /t 0
```
**What:** `/fw` boots to UEFI/BIOS setup. `/o` boots to Advanced Startup (WinRE).
**Why:** No more mashing F2/Del or hunting for the Shift+Restart trick. **Save your work first: it's immediate.**
**Danger:** 🟡 Moderate

#### 28. Inspect Boot Configuration Data
```cmd
bcdedit /enum all /v
```
**What:** Dumps every BCD entry (Admin).
**Why:** Diagnose dual-boot, safe-mode or hypervisor launch flags. Read-only until you use `/set`.
**Danger:** 🟢 Safe *(editing BCD is 🔴)*

---

# 4. Reports

#### 29. DirectX/GPU/Display Diagnostic Dump
```powershell
dxdiag /whql:off /t "$HOME\Desktop\dxdiag.txt"
```
**What:** Writes GPU, driver, monitor and audio info to a text file.
**Why:** The artifact every game/driver support forum asks for.
**Danger:** 🟢 Safe

#### 30. Wi-Fi History and Failure Report
```cmd
netsh wlan show wlanreport
```
**What:** Builds an HTML report of the last 3 days of Wi-Fi sessions, disconnect reasons and signal quality (Admin). Output: `C:\ProgramData\Microsoft\Windows\WlanReport\wlan-report-latest.html`.
**Why:** Proves a flaky access point or a driver problem with timestamps.
**Danger:** 🟢 Safe

#### 31. Readable Windows Update Log
```powershell
Get-WindowsUpdateLog -LogPath "$HOME\Desktop\WindowsUpdate.log"
```
**What:** Converts binary ETL update traces into a text log.
**Why:** Only way to see why an update fails (error codes, download source, retry logic).
**Danger:** 🟢 Safe

#### 32. Unsigned or Problem Drivers
```powershell
driverquery /si /fo csv | ConvertFrom-Csv | ? IsSigned -notmatch 'TRUE'
```
**What:** Filters the signed-driver inventory down to unsigned entries.
**Why:** Unsigned drivers are a common crash and security finding.
**Danger:** 🟢 Safe

#### 33. Targeted Machine Facts via Get-ComputerInfo
```powershell
Get-ComputerInfo OsName,OsVersion,OsBuildNumber,WindowsInstallDateFromRegistry,BiosSMBIOSBIOSVersion,CsTotalPhysicalMemory,HyperVisorPresent,DeviceGuardSecurityServicesRunning
```
**What:** Requests only specific properties. The full call is slow.
**Why:** Fast, scriptable fleet inventory. Note DeviceGuard shows VBS/HVCI state.
**Danger:** 🟢 Safe

#### 34. Last 10 Installed Updates
```powershell
Get-HotFix | sort InstalledOn -Desc | select -First 10 HotFixID,Description,InstalledOn,InstalledBy
```
**What:** Recent KBs with dates.
**Why:** "It broke after Tuesday" → here's the culprit KB for `wusa /uninstall /kb:…`.
**Danger:** 🟢 Safe

#### 35. Instant HTML Report From Any Command
```powershell
Get-Service | Select Name,Status,StartType | ConvertTo-Html -Title 'Services' -PreContent '<h1>Services</h1>' | Out-File "$HOME\Desktop\services.html"; ii "$HOME\Desktop\services.html"
```
**What:** Pipes any objects into a self-contained HTML table and opens it.
**Why:** Shareable reports with zero dependencies. Add `-CssUri` for styling.
**Danger:** 🟢 Safe

#### 36. See Which Services Hide Inside Each svchost
```cmd
tasklist /svc /fi "imagename eq svchost.exe"
```
**What:** Maps each `svchost.exe` PID to the services it hosts.
**Why:** Turns the "why is svchost at 40% CPU?" mystery into a named service.
**Danger:** 🟢 Safe

---

# 5. Networking

#### 37. Recover Every Saved Wi-Fi Password
```powershell
netsh wlan show profiles | sls 'All User Profile' | % { $n=($_ -split ':',2)[1].Trim(); netsh wlan show profile name="$n" key=clear } | sls 'SSID name|Key Content'
```
**What:** Prints each stored SSID with its cleartext key (Admin).
**Why:** Recover a forgotten Wi-Fi password or migrate settings. Only run on machines you own.
**Danger:** 🟡 Moderate *(exposes secrets on screen and in history)*

#### 38. Wi-Fi Channel Survey (No App Needed)
```cmd
netsh wlan show networks mode=bssid
```
**What:** Lists every visible AP with BSSID, channel, band, radio type and signal %.
**Why:** Pick the least crowded channel for your router.
**Danger:** 🟢 Safe

#### 39. Packet Capture Without Wireshark
```cmd
pktmon start --capture --comp nics
pktmon stop
pktmon pcapng PktMon.etl -o capture.pcapng
```
```cmd
:: Classic alternative
netsh trace start capture=yes report=yes maxsize=512 tracefile=C:\net.etl
netsh trace stop
```
**What:** Built-in packet capture (Admin). `pktmon` also converts to `.pcapng` for Wireshark; older builds use `pktmon etl2pcap`.
**Why:** Capture on locked-down servers where you can't install tools.
**Danger:** 🟡 Moderate *(captures sensitive traffic; large files)*

#### 40. Test a TCP Port With Full Diagnostics
```powershell
Test-NetConnection example.com -Port 443 -InformationLevel Detailed
Test-NetConnection example.com -TraceRoute
```
**What:** DNS resolution, source interface, route, TCP result and remote address in one object.
**Why:** Replaces `ping`, `telnet` and `tracert` in one cmdlet.
**Danger:** 🟢 Safe

#### 41. Every Connection With the Owning Process
```powershell
Get-NetTCPConnection -State Established | Select RemoteAddress,RemotePort,@{n='Process';e={(Get-Process -Id $_.OwningProcess -EA 0).Name}} | sort Process | ft -Auto
```
**What:** Maps live connections to process names.
**Why:** Spot unexpected outbound connections without `netstat -b` parsing.
**Danger:** 🟢 Safe

#### 42. DNS Queries Against a Chosen Resolver
```powershell
Resolve-DnsName example.com -Type MX -Server 1.1.1.1 -DnsOnly
Get-DnsClientCache | Select -First 20 Entry,Type,Data
```
**What:** Query any record type against any resolver, bypassing the hosts file and LLMNR. Also dumps the local DNS cache.
**Why:** Compare internal vs public answers when debugging split-DNS or propagation.
**Danger:** 🟢 Safe

#### 43. Find Ports Windows Has Silently Reserved
```cmd
netsh interface ipv4 show excludedportrange protocol=tcp
```
**What:** Lists port ranges reserved by Hyper-V, WSL, Docker and Windows NAT.
**Why:** The fix for "port already in use" or "access permission denied" when nothing is listening.
**Danger:** 🟢 Safe

#### 44. Built-in Port Forwarding
```cmd
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=192.168.1.10
netsh interface portproxy show all
netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=0.0.0.0
```
**What:** Kernel-level TCP port forwarding (Admin).
**Why:** Expose a WSL2 or VM service to your LAN without installing a proxy.
**Danger:** 🟡 Moderate *(opens a listener; pair with a firewall rule)*

#### 45. PathPing: Packet Loss Per Hop
```cmd
pathping -q 50 -w 250 8.8.8.8
```
**What:** Traces the route, then sends 50 probes per hop and reports loss and latency per router.
**Why:** Finds *which* hop is dropping packets. `tracert` can't do that.
**Danger:** 🟢 Safe *(takes a few minutes)*

#### 46. Latency Breakdown for Any URL (DNS / TCP / TLS / TTFB)
```powershell
curl.exe -o NUL -s -w 'DNS %{time_namelookup}s | TCP %{time_connect}s | TLS %{time_appconnect}s | TTFB %{time_starttransfer}s | Total %{time_total}s\n' https://example.com
```
**What:** Uses the `curl.exe` bundled with Windows to split request time into phases.
**Why:** Tells you whether slowness is DNS, the network, TLS or the server.
**Danger:** 🟢 Safe

#### 47. Log Dropped Packets in Windows Firewall
```cmd
netsh advfirewall set currentprofile logging droppedconnections enable
netsh advfirewall set currentprofile logging filemaxsize 16384
```
```powershell
gc "$env:windir\System32\LogFiles\Firewall\pfirewall.log" -Wait -Tail 20
```
**What:** Enables logging of blocked traffic and tails the log.
**Why:** Shows exactly which rule-less traffic is being dropped when an app "can't connect". Disable afterward to avoid growth.
**Danger:** 🟡 Moderate

#### 48. WinHTTP Proxy: The Setting GUIs Don't Show
```cmd
netsh winhttp show proxy
netsh winhttp import proxy source=ie
netsh winhttp reset proxy
```
**What:** Views and sets the machine-wide proxy used by services and system components (Windows Update, BITS).
**Why:** Fixes "browser works, Windows Update doesn't" behind proxies.
**Danger:** 🟡 Moderate

---

# 6. Repair & maintenance

#### 49. The Correct DISM Repair Sequence
```cmd
DISM /Online /Cleanup-Image /CheckHealth
DISM /Online /Cleanup-Image /ScanHealth
DISM /Online /Cleanup-Image /RestoreHealth
:: offline source if Windows Update is broken:
DISM /Online /Cleanup-Image /RestoreHealth /Source:WIM:D:\sources\install.wim:1 /LimitAccess
```
**What:** Quick check → deep scan → repair of the component store (Admin).
**Why:** Run **before** `sfc /scannow`; SFC pulls good files from the store DISM repairs.
**Danger:** 🟡 Moderate

#### 50. See Which Files SFC Actually Repaired
```powershell
sfc /scannow
sls '\[SR\]' "$env:windir\Logs\CBS\CBS.log" | select -Last 40
```
**What:** Extracts SFC's own entries from the CBS log.
**Why:** "Windows Resource Protection found corrupt files" is useless without the file names.
**Danger:** 🟢 Safe

#### 51. Online CHKDSK Without a Reboot
```cmd
chkdsk C: /scan
chkdsk C: /spotfix
```
**What:** `/scan` checks the live NTFS volume. `/spotfix` fixes only the logged issues at next boot in seconds.
**Why:** Avoid the 2-hour full `/f /r` on big volumes.
**Danger:** 🟡 Moderate

#### 52. TRIM / Optimize From PowerShell
```powershell
Optimize-Volume -DriveLetter C -Analyze -Verbose
Optimize-Volume -DriveLetter C -ReTrim -Verbose
```
**What:** Analyzes fragmentation and re-issues TRIM to SSDs.
**Why:** Proper maintenance for SSDs (never defrag them) with clear output.
**Danger:** 🟢 Safe

#### 53. List and Enable Optional Windows Features
```cmd
DISM /Online /Get-Features /Format:Table
DISM /Online /Enable-Feature /FeatureName:Microsoft-Hyper-V-All /All /NoRestart
```
**What:** Enumerates and toggles features like Hyper-V, Sandbox, WSL, SMB1, Telnet.
**Why:** Script a full dev-box configuration, or hunt features you should disable (SMB1).
**Danger:** 🟡 Moderate

#### 54. Install OpenSSH Server (and RSAT) Offline-Style
```cmd
DISM /Online /Get-Capabilities /Format:Table
DISM /Online /Add-Capability /CapabilityName:OpenSSH.Server~~~~0.0.1.0
```
```powershell
Start-Service sshd; Set-Service sshd -StartupType Automatic
```
**What:** Features-on-Demand are "capabilities": OpenSSH, RSAT, Language packs.
**Why:** Turn Windows into an SSH target in 30 seconds. Harden `sshd_config` afterward.
**Danger:** 🟡 Moderate

#### 55. Robocopy: The Real Mirror Command
```cmd
robocopy C:\Src D:\Backup /MIR /MT:32 /R:1 /W:1 /XD node_modules .git /NP /LOG+:C:\logs\copy.log
```
**What:** Multithreaded mirror that skips junk, retries once, and logs.
**Why:** Faster and more reliable than Explorer copy. **`/MIR` deletes extras in the destination**; run with `/L` first for a dry run.
**Danger:** 🟡 Moderate

#### 56. Reset Broken NTFS Permissions
```cmd
takeown /f D:\Data /r /d y
icacls D:\Data /reset /t /c /q
```
**What:** Takes ownership, then replaces all ACLs with inherited defaults.
**Why:** Fixes "access denied" on drives moved from another PC. Never run on system folders.
**Danger:** 🔴 Advanced

---

# 7. PowerCfg

#### 57. Battery Health Report
```cmd
powercfg /batteryreport /duration 14 /output "%USERPROFILE%\Desktop\battery.html"
```
**What:** Design vs full-charge capacity, cycle count and 14 days of usage.
**Why:** Know when to replace a laptop battery.
**Danger:** 🟢 Safe

#### 58. Energy Efficiency Diagnostics
```cmd
powercfg /energy /duration 60 /output "%USERPROFILE%\Desktop\energy.html"
```
**What:** Observes the system for 60 seconds (leave it idle; Admin) and flags errors in power policy, USB suspend, timer resolution and more.
**Why:** Identifies drivers or apps that keep the CPU awake and drain batteries.
**Danger:** 🟢 Safe

#### 59. Modern Standby Drain Report
```cmd
powercfg /sleepstudy /duration 7 /output "%USERPROFILE%\Desktop\sleepstudy.html"
```
**What:** Per-session drain during sleep, with the top offending apps/devices.
**Why:** Fixes laptops that lose 30% overnight. Requires a Modern Standby (S0) device.
**Danger:** 🟢 Safe

#### 60. What Is Preventing Sleep Right Now?
```cmd
powercfg /requests
```
**What:** Lists active display/system/awaymode requests by process and driver.
**Why:** Instantly names the app that stops your PC sleeping.
**Danger:** 🟢 Safe

#### 61. What Woke My PC?
```cmd
powercfg /lastwake
powercfg /devicequery wake_armed
```
**What:** Reports the last wake source and lists devices allowed to wake the machine.
**Why:** Stop random 3 AM wake-ups by disabling "allow this device to wake" on the culprit.
**Danger:** 🟢 Safe

#### 62. Unlock the Ultimate Performance Power Plan
```cmd
powercfg -duplicatescheme e9a42b02-d5df-448d-aa00-03f14749eb61
powercfg /list
```
**What:** Clones the hidden plan; activate it with `powercfg /setactive <new-GUID>`.
**Why:** Removes micro-latency from core parking and throttling for workstations and audio work. Increases power draw and heat.
**Danger:** 🟡 Moderate

#### 63. Dump and Unhide Every Power Setting
```cmd
powercfg /qh > "%USERPROFILE%\Desktop\hidden-power-settings.txt"
powercfg -attributes SUB_PROCESSOR be337238-0d82-4146-a960-4f3749d470c7 -ATTRIB_HIDE
```
**What:** `/qh` lists all settings including hidden ones. The second command unhides *Processor performance boost mode* in Power Options.
**Why:** Reveals hundreds of tunables Microsoft hides from the GUI.
**Danger:** 🟡 Moderate

#### 64. Per-App Energy Usage From SRUM
```cmd
powercfg /srumutil /output "%USERPROFILE%\Desktop\srum.csv" /csv
```
**What:** Exports the System Resource Usage Monitor database (Admin): per-app CPU, network and energy for the past ~30 days.
**Why:** Forensic-quality history of what ran and how heavily, even for closed apps.
**Danger:** 🟢 Safe

---

# 8. Sysinternals

#### 65. Run Sysinternals Tools Straight From the Web
```powershell
\\live.sysinternals.com\tools\procexp64.exe
winget install Microsoft.Sysinternals.Suite
```
**What:** Executes tools from Microsoft's WebDAV share with no download (needs the *WebClient* service and network access).
**Why:** Troubleshoot a client machine without installing anything. Winget installs the whole suite permanently.
**Danger:** 🟢 Safe

#### 66. Auto-Capture Crash and CPU-Spike Dumps
```cmd
procdump -accepteula -ma -e -w myapp.exe C:\dumps
procdump -accepteula -ma -c 90 -s 5 -n 3 myapp.exe C:\dumps
```
**What:** Waits for the app and dumps it on an unhandled exception, or when CPU stays ≥90% for 5 seconds (up to 3 dumps).
**Why:** Catch intermittent bugs without a debugger attached. Dumps may contain secrets.
**Danger:** 🟡 Moderate

#### 67. Who Has This File Locked?
```cmd
handle64 -accepteula -nobanner "C:\Users\me\Documents\locked.xlsx"
```
**What:** Lists processes holding open handles that match the path (Admin).
**Why:** Ends the "file in use by another process" guessing game.
**Danger:** 🟢 Safe *(closing handles with `-c` is 🔴)*

#### 68. Folder Sizes at Depth, in the Terminal
```cmd
du64 -accepteula -l 2 -q C:\Users
```
**What:** Recursive disk usage to a chosen depth.
**Why:** Fast, scriptable TreeSize alternative that counts hardlinks once.
**Danger:** 🟢 Safe

#### 69. Open a SYSTEM-Level Shell
```cmd
psexec64 -accepteula -s -i -d powershell.exe
```
**What:** Launches PowerShell as `NT AUTHORITY\SYSTEM` on the interactive desktop.
**Why:** Inspect protected registry hives and service-owned files exactly as services see them. Easy to damage a system from here.
**Danger:** 🔴 Advanced

#### 70. Autoruns From the Command Line
```cmd
autorunsc64 -accepteula -a * -c -h -s -nobanner > autoruns.csv
```
**What:** Exports every persistence location (Run keys, tasks, services, drivers, WMI, shell extensions) with hashes and signature verification.
**Why:** The best malware-hunting and boot-cleanup baseline on Windows. Diff exports over time.
**Danger:** 🟢 Safe

#### 71. Hunt Unsigned Binaries
```cmd
sigcheck64 -accepteula -u -e -s C:\Windows\System32
```
**What:** Recursively lists unsigned executables.
**Why:** A fast integrity sweep. Unsigned ≠ malicious, but is worth reviewing.
**Danger:** 🟢 Safe

#### 72. Headless Process Monitor Capture
```cmd
procmon64 /AcceptEula /Quiet /Minimized /BackingFile C:\pm.pml
procmon64 /Terminate
```
**What:** Starts a Procmon trace with no UI and ends it when you're done.
**Why:** Capture an elusive registry/file/process failure on a remote box, then analyze the `.pml` on your PC.
**Danger:** 🟡 Moderate *(large logs; may contain sensitive paths)*

---

# 9. Windows Sandbox & Hyper-V

#### 73. Windows Sandbox: Disposable Desktop With a .wsb Config
```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Containers-DisposableClientVM -All -NoRestart
```
```xml
<!-- C:\Sandbox\analysis.wsb -->
<Configuration>
  <VGpu>Disable</VGpu>
  <Networking>Disable</Networking>
  <MappedFolders>
    <MappedFolder>
      <HostFolder>C:\Samples</HostFolder>
      <SandboxFolder>C:\Samples</SandboxFolder>
      <ReadOnly>true</ReadOnly>
    </MappedFolder>
  </MappedFolders>
  <LogonCommand><Command>explorer.exe C:\Samples</Command></LogonCommand>
</Configuration>
```
```cmd
WindowsSandbox.exe C:\Sandbox\analysis.wsb
```
**What:** A throwaway, network-isolated Windows desktop with a read-only shared folder. Everything vanishes on close.
**Why:** Open suspicious files or test installers safely. Requires Pro/Enterprise/Education + virtualization.
**Danger:** 🟢 Safe

#### 74. Windows Sandbox CLI (`wsb`)
```cmd
wsb start
wsb list
wsb stop --id <ID>
wsb --help
```
**What:** Newer Windows 11 builds (24H2+) add a CLI to create, list, stop, and execute commands inside sandboxes.
**Why:** Scriptable disposable test environments for CI-like testing of installers and scripts. Check `wsb --help` for your build's verbs.
**Danger:** 🟢 Safe

#### 75. Hyper-V Fleet Overview
```powershell
Get-VM | Select Name,State,@{n='CPU%';e={$_.CPUUsage}},@{n='MemGB';e={[math]::Round($_.MemoryAssigned/1GB,1)}},Uptime,Status
```
**What:** One-table view of every VM's state, load, memory and uptime.
**Why:** A lightweight dashboard without Hyper-V Manager.
**Danger:** 🟢 Safe

#### 76. Snapshot, Patch, Roll Back
```powershell
Checkpoint-VM -Name dev01 -SnapshotName 'pre-patch'
# ... test something risky ...
Restore-VMCheckpoint -VMName dev01 -Name 'pre-patch' -Confirm:$false
```
**What:** Creates and restores a VM checkpoint.
**Why:** Fearless experimentation. Don't use checkpoints on domain controllers or as long-term backups.
**Danger:** 🟡 Moderate

#### 77. NAT Network for VMs (Built-In, No Router Config)
```powershell
New-VMSwitch -SwitchName 'NAT-Lab' -SwitchType Internal
New-NetIPAddress -IPAddress 192.168.100.1 -PrefixLength 24 -InterfaceAlias 'vEthernet (NAT-Lab)'
New-NetNat -Name LabNAT -InternalIPInterfaceAddressPrefix 192.168.100.0/24
```
**What:** Builds an isolated subnet with outbound internet via NAT for guests.
**Why:** Reproducible lab networking that doesn't touch your LAN. Give guests static IPs in `192.168.100.0/24`, gateway `.1`.
**Danger:** 🔴 Advanced

#### 78. Mount a VHDX on the Host, Then Shrink It
```powershell
Mount-VHD -Path D:\vm\dev.vhdx -ReadOnly -Passthru | Get-Disk | Get-Partition
Dismount-VHD -Path D:\vm\dev.vhdx
Optimize-VHD -Path D:\vm\dev.vhdx -Mode Full      # VM must be off
```
**What:** Browse a VM's disk from the host and reclaim unused space.
**Why:** Rescue files from a VM that won't boot; shrink bloated dynamic disks (zero free space first with `sdelete -z`).
**Danger:** 🟡 Moderate

---

# 10. Task Scheduler & Event Viewer

#### 79. One-Line Scheduled Task as SYSTEM
```cmd
schtasks /create /tn "Maint\WeeklyCleanup" /tr "powershell.exe -NoProfile -ExecutionPolicy Bypass -File C:\Scripts\Cleanup.ps1" /sc weekly /d SUN /st 03:00 /ru SYSTEM /rl HIGHEST /f
```
**What:** Creates a high-privilege weekly job in a custom folder.
**Why:** Deploy maintenance without clicking through the wizard. Secure the script folder (ACLs) as it runs as SYSTEM.
**Danger:** 🟡 Moderate

#### 80. Robust Scheduled Task via PowerShell
```powershell
$a = New-ScheduledTaskAction -Execute 'pwsh.exe' -Argument '-NoProfile -File C:\Scripts\Report.ps1'
$t = New-ScheduledTaskTrigger -Daily -At 7am
$s = New-ScheduledTaskSettingsSet -StartWhenAvailable -RunOnlyIfNetworkAvailable -ExecutionTimeLimit (New-TimeSpan -Hours 1)
Register-ScheduledTask -TaskName 'DailyReport' -Action $a -Trigger $t -Settings $s -User 'SYSTEM' -RunLevel Highest
```
**What:** Adds "run if missed", network condition and a timeout.
**Why:** The settings that make tasks reliable on laptops that sleep.
**Danger:** 🟡 Moderate

#### 81. Audit Every Non-Microsoft Scheduled Task
```powershell
Get-ScheduledTask | ? { $_.State -ne 'Disabled' -and $_.TaskPath -notlike '\Microsoft*' } | % { $i = $_ | Get-ScheduledTaskInfo; [pscustomobject]@{Task=$_.TaskName;Path=$_.TaskPath;Last=$i.LastRunTime;Result=$i.LastTaskResult;Action=($_.Actions.Execute+' '+$_.Actions.Arguments)} } | Out-GridView
```
**What:** Shows third-party tasks with last run, result and command line.
**Why:** Where updaters, malware and forgotten scripts hide.
**Danger:** 🟢 Safe

#### 82. Trigger a Task When an Event Is Logged
```cmd
schtasks /create /tn "OnNetworkConnect" /tr "powershell -File C:\Scripts\OnNet.ps1" /sc ONEVENT /ec "Microsoft-Windows-NetworkProfile/Operational" /mo "*[System[(EventID=10000)]]" /ru SYSTEM
```
**What:** Runs a script whenever Windows connects to a network (Event 10000).
**Why:** Event-driven automation: mount shares, switch VPN, or log SSID on connect. Works with any log/ID.
**Danger:** 🟡 Moderate

#### 83. Last-24-Hour Errors With XPath
```cmd
wevtutil qe System /q:"*[System[(Level=1 or Level=2) and TimeCreated[timediff(@SystemTime) <= 86400000]]]" /c:25 /rd:true /f:text
```
**What:** Server-side filtered query: critical/error events from the last day (86,400,000 ms), newest first.
**Why:** Far faster than loading the Event Viewer GUI.
**Danger:** 🟢 Safe

#### 84. Who Is Failing to Log In?
```powershell
Get-WinEvent -FilterHashtable @{LogName='Security';Id=4625} -MaxEvents 500 |
 % { [pscustomobject]@{User=$_.Properties[5].Value;IP=$_.Properties[19].Value} } | group User,IP | sort Count -Desc | select -First 10 Count,Name
```
**What:** Aggregates failed logons by account and source IP (Admin).
**Why:** Spots brute-force against RDP/SMB. Property indexes apply to 4625 on current Windows.
**Danger:** 🟢 Safe

#### 85. Why Did My PC Reboot?
```powershell
Get-WinEvent -FilterHashtable @{LogName='System';Id=41,1074,6005,6006,6008} -MaxEvents 15 | select TimeCreated,Id,Message | fl
```
**What:** Kernel-Power 41 (crash/power loss), 1074 (who initiated), 6008 (unexpected shutdown), 6005/6006 (log start/stop).
**Why:** Distinguishes updates, user action and genuine crashes.
**Danger:** 🟢 Safe

#### 86. Find Noisy and Oversized Event Logs
```powershell
Get-WinEvent -ListLog * -EA 0 | ? RecordCount | sort RecordCount -Desc | select -First 15 LogName,RecordCount,@{n='SizeMB';e={[math]::Round($_.FileSize/1MB,1)}}
```
**What:** Ranks logs by record count and size.
**Why:** A log flooding with thousands of events/hour is usually your broken component.
**Danger:** 🟢 Safe

---

# 11. Registry tweaks

> Back up first: `reg export HKCU C:\hkcu-backup.reg`, or create a restore point.

#### 87. Restore the Classic Context Menu (Windows 11)
```cmd
reg add "HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}\InprocServer32" /ve /d "" /f
taskkill /f /im explorer.exe & start explorer.exe
```
**What:** Skips the "Show more options" submenu.
**Why:** Full right-click menu immediately. Undo with `reg delete "HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}" /f`.
**Danger:** 🟡 Moderate

#### 88. Disable Web Results in Start Search
```cmd
reg add "HKCU\Software\Policies\Microsoft\Windows\Explorer" /v DisableSearchBoxSuggestions /t REG_DWORD /d 1 /f
```
**What:** Turns off Bing/web suggestions in Start search.
**Why:** Faster, local-only, private search. Sign out and back in.
**Danger:** 🟢 Safe

#### 89. Remove the 260-Character Path Limit
```cmd
reg add HKLM\SYSTEM\CurrentControlSet\Control\FileSystem /v LongPathsEnabled /t REG_DWORD /d 1 /f
```
**What:** Enables Win32 long paths (up to ~32,767 chars) for apps that opt in.
**Why:** Fixes `node_modules`/Git/build tooling failures. Needs Admin and a restart.
**Danger:** 🟡 Moderate

#### 90. Verbose Logon/Logoff/Shutdown Messages
```cmd
reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v VerboseStatus /t REG_DWORD /d 1 /f
```
**What:** Shows what Windows is doing at boot ("Applying computer settings", "Waiting for…").
**Why:** Turns a hanging "Welcome" screen into an actionable clue.
**Danger:** 🟢 Safe

#### 91. Fix Dual-Boot Clock Drift
```cmd
reg add HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation /v RealTimeIsUniversal /t REG_QWORD /d 1 /f
```
**What:** Tells Windows the hardware clock is UTC (like Linux/macOS).
**Why:** Ends the "time is off by N hours after switching OS" issue. Alternative: set Linux to localtime.
**Danger:** 🟡 Moderate

#### 92. "End Task" in the Taskbar Right-Click Menu (Windows 11)
```cmd
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced\TaskbarDeveloperSettings" /v TaskbarEndTask /t REG_DWORD /d 1 /f
```
**What:** Adds *End task* to taskbar app context menus. Also in Settings → System → For developers on supporting builds.
**Why:** Kill frozen apps in one click.
**Danger:** 🟢 Safe

#### 93. Diff the Registry Before and After Any Change
```powershell
reg export HKCU\Software before.reg /y
# ... change a setting in the UI or install an app ...
reg export HKCU\Software after.reg /y
Compare-Object (gc before.reg) (gc after.reg) | ? SideIndicator -eq '=>' | select -Expand InputObject
```
**What:** Reveals exactly which keys/values a GUI toggle or installer changed.
**Why:** The best way to reverse-engineer settings, then script them for deployment.
**Danger:** 🟢 Safe

---

# 12. Winget, Terminal & productivity

#### 94. Export and Re-Create Your Entire App Set
```powershell
winget export -o apps.json --include-versions
winget import -i apps.json --accept-package-agreements --accept-source-agreements
```
**What:** Writes installed packages to JSON and reinstalls them on another machine.
**Why:** Rebuild a PC in minutes. Commit `apps.json` to your dotfiles repo.
**Danger:** 🟡 Moderate

#### 95. Upgrade Everything, Pin What You Must Not Touch
```powershell
winget pin add --id Mozilla.Firefox
winget upgrade --all --include-unknown --silent --accept-package-agreements --accept-source-agreements
```
**What:** Updates all apps in one go while excluding pinned ones.
**Why:** Patch management for personal machines without surprises.
**Danger:** 🟡 Moderate

#### 96. Script Windows Terminal Layouts
```cmd
wt -d C:\src new-tab -p "PowerShell" ; split-pane -V -d C:\logs ; new-tab -p "Ubuntu"
wt -w _quake
```
**What:** `wt` command-line arguments build tabs and panes from a shell, shortcut or script. `-w _quake` opens the dropdown "Quake mode" terminal (also **Win+`**).
**Why:** One shortcut opens your whole dev workspace.
**Danger:** 🟢 Safe

#### 97. Fish-Style Predictive Command Line
```powershell
Set-PSReadLineOption -PredictionSource HistoryAndPlugin -PredictionViewStyle ListView
# Press F2 to toggle inline/list view, Ctrl+R for history search
```
**What:** Shows history-based predictions as you type. Add to `$PROFILE`.
**Why:** Massive speed-up for repetitive commands. `Install-Module CompletionPredictor` adds argument completions.
**Danger:** 🟢 Safe

#### 98. Create a ZIP With the Built-in `tar`
```cmd
tar -a -c -f backup.zip MyFolder
tar -xf backup.zip -C D:\restore
```
**What:** Windows ships `bsdtar`, which picks the format by extension.
**Why:** No 7-Zip needed, and it's faster than `Compress-Archive`. Also handles `.tar.gz`, `.7z` (read) and more.
**Danger:** 🟢 Safe

#### 99. Move Heavy Folders to Another Drive With a Junction
```cmd
robocopy "C:\Users\me\AppData\Local\BigCache" "D:\BigCache" /MIR /COPYALL /R:1 /W:1
rmdir /s /q "C:\Users\me\AppData\Local\BigCache"
mklink /J "C:\Users\me\AppData\Local\BigCache" "D:\BigCache"
```
**What:** Relocates a folder transparently, and the app still sees the old path.
**Why:** Rescue a full C: drive without reinstalling apps. Close the app first. **`rmdir /s /q` is destructive; verify the copy.**
**Danger:** 🔴 Advanced

#### 100. Dev Drive: ReFS With Async Defender for Fast Builds
```powershell
Format-Volume -DriveLetter E -DevDrive
fsutil devdrv query E:
```
**What:** Formats a volume (≥50 GB, **erases it**) as a Dev Drive: ReFS tuned for developer workloads with performance-mode antivirus (Windows 11 23H2+).
**Why:** Noticeably faster `git`, `npm`, `cargo`, and MSBuild on large repos.
**Danger:** 🔴 Advanced

---

# 🧪 20 PowerShell One-Liners

#### PS-01 · `tail -f` for Windows
```powershell
Get-Content app.log -Wait -Tail 50
```
**What:** Streams new lines as they're written. **Why:** Live log watching with no extra tools; add `| sls ERROR` to filter. **Danger:** 🟢 Safe

#### PS-02 · Cryptographically Random Password (PS7)
```powershell
[Convert]::ToBase64String([Security.Cryptography.RandomNumberGenerator]::GetBytes(24))
```
**What:** 32-character secret from the OS CSPRNG. **Why:** `Get-Random` is *not* cryptographically secure; this is. **Danger:** 🟢 Safe

#### PS-03 · Parallel Ping Sweep (PS7)
```powershell
1..254 | % -Parallel { $ip="192.168.1.$_"; if (Test-Connection $ip -Count 1 -TimeoutSeconds 1 -Quiet) { $ip } } -ThrottleLimit 100
```
**What:** Finds live hosts on a /24 in seconds. **Why:** `ForEach-Object -Parallel` makes network discovery trivial. Scan only networks you own. **Danger:** 🟡 Moderate

#### PS-04 · TCP Port Scanner in One Line (PS7)
```powershell
$t='192.168.1.1'; 1..1024 | % -Parallel { $c=[Net.Sockets.TcpClient]::new(); if ($c.ConnectAsync($using:t,$_).Wait(200)) { $_ }; $c.Dispose() } -ThrottleLimit 200
```
**What:** Reports open ports using .NET sockets. **Why:** Check your own firewall posture without installing nmap. **Danger:** 🟡 Moderate

#### PS-05 · Who Is a Local Administrator?
```powershell
Get-LocalGroupMember Administrators | select Name,ObjectClass,PrincipalSource
```
**What:** Lists admin group members, including domain/Azure AD principals. **Why:** First step in any privilege audit. **Danger:** 🟢 Safe

#### PS-06 · What Changed in the Last 24 Hours?
```powershell
gci C:\Users\$env:USERNAME -Recurse -File -EA 0 | ? LastWriteTime -gt (Get-Date).AddDays(-1) | sort LastWriteTime -Desc | select -First 30 LastWriteTime,FullName
```
**What:** Recently modified files. **Why:** Find where a rogue app is writing, or that file you just saved "somewhere". **Danger:** 🟢 Safe

#### PS-07 · Compare Two Directory Trees
```powershell
Compare-Object (gci A:\ -Recurse -File) (gci B:\ -Recurse -File) -Property Name,Length
```
**What:** Shows files missing or different in size between two trees. **Why:** Quick verification of a backup or deployment. **Danger:** 🟢 Safe

#### PS-08 · Latest Release of Any GitHub Project
```powershell
irm https://api.github.com/repos/PowerShell/PowerShell/releases/latest | select tag_name,published_at,html_url
```
**What:** Calls a REST API and returns parsed objects. **Why:** Native JSON handling with no `jq`. Unauthenticated calls are rate-limited. **Danger:** 🟢 Safe

#### PS-09 · Drivers Sorted by Age
```powershell
Get-WindowsDriver -Online -All | sort Date | select -First 20 Driver,OriginalFileName,ProviderName,Version,Date
```
**What:** Third-party driver packages, oldest first (Admin). **Why:** Old drivers are prime suspects for BSODs and security findings. **Danger:** 🟢 Safe

#### PS-10 · Process Command Lines (The Truth About "chrome.exe")
```powershell
Get-CimInstance Win32_Process | ? CommandLine | select ProcessId,ParentProcessId,Name,CommandLine | Out-GridView
```
**What:** Full command line and parent PID for every process. **Why:** Reveals encoded PowerShell, injected flags and which Java app is *actually* running. **Danger:** 🟢 Safe

#### PS-11 · Free Space as a Percentage
```powershell
Get-Volume | ? DriveLetter | select DriveLetter,FileSystemLabel,@{n='FreeGB';e={[math]::Round($_.SizeRemaining/1GB,1)}},@{n='Free%';e={[math]::Round($_.SizeRemaining/$_.Size*100,1)}}
```
**What:** Clean capacity summary. **Why:** Readable at a glance and trivially alertable. **Danger:** 🟢 Safe

#### PS-12 · Base64 Round Trip
```powershell
$b=[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes('hello')); [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($b))
```
**What:** Encode/decode text. **Why:** Debug tokens, `-EncodedCommand` payloads and configs. **Danger:** 🟢 Safe

#### PS-13 · Time Any Command
```powershell
Measure-Command { robocopy C:\Src D:\Dst /E /MT:32 | Out-Null } | select TotalSeconds
```
**What:** Wall-clock timing. **Why:** Benchmark flags and approaches honestly. **Danger:** 🟢 Safe

#### PS-14 · Bulk Rename With a Script Block
```powershell
gci *.jpeg | Rename-Item -NewName { $_.Name -replace '\.jpeg$','.jpg' } -WhatIf
```
**What:** Regex renames; remove `-WhatIf` to apply. **Why:** `-WhatIf` is the most underused safety net in PowerShell. **Danger:** 🟡 Moderate

#### PS-15 · Which Process Owns Port 3000?
```powershell
Get-Process -Id (Get-NetTCPConnection -LocalPort 3000 -State Listen).OwningProcess | select Id,Name,Path
```
**What:** Maps a listening port to its executable. **Why:** "EADDRINUSE" solved in one line. **Danger:** 🟢 Safe

#### PS-16 · Certificates Expiring Within 60 Days
```powershell
gci Cert:\LocalMachine\My | ? NotAfter -lt (Get-Date).AddDays(60) | select Subject,NotAfter,Thumbprint | sort NotAfter
```
**What:** Machine certificate expiry watchlist. **Why:** Prevents 2 AM outages; works across `Cert:\CurrentUser` too. **Danger:** 🟢 Safe

#### PS-17 · Installed Software From the Registry (Fast, Unlike Win32_Product)
```powershell
gp HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*,HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* -EA 0 | ? DisplayName | select DisplayName,DisplayVersion,Publisher,InstallDate | sort DisplayName
```
**What:** Inventory of 32- and 64-bit apps. **Why:** `Win32_Product` triggers MSI repair on every package; this doesn't. **Danger:** 🟢 Safe

#### PS-18 · Unix Epoch Time
```powershell
[DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
```
**What:** Current Unix timestamp. **Why:** Needed for JWTs, logs and APIs. Convert back with `[DateTimeOffset]::FromUnixTimeSeconds(…)`. **Danger:** 🟢 Safe

#### PS-19 · grep With Context
```powershell
sls -Path .\*.log -Pattern 'ERROR|FATAL' -Context 2,3 | select -First 10
```
**What:** Matches plus 2 lines before and 3 after. **Why:** `grep -B/-A` for Windows. **Danger:** 🟢 Safe

#### PS-20 · Relaunch Elevated
```powershell
Start-Process pwsh -Verb RunAs -ArgumentList '-NoExit','-Command','cd',"'$PWD'"
```
**What:** Opens an admin PowerShell in the current directory. **Why:** `sudo` muscle memory without leaving the terminal. **Danger:** 🟡 Moderate

---

# 🔍 20 Hidden Windows Tools

#### HT-01 · God Mode (All Control Panel Tasks)
```cmd
explorer shell:::{ED7BA470-8E54-465E-825C-99712043E01C}
```
**What:** Opens a flat list of ~200 admin tasks (or create a folder named `GodMode.{ED7BA470-8E54-465E-825C-99712043E01C}`). **Why:** Search any setting instantly. **Danger:** 🟢 Safe

#### HT-02 · IExpress: Build Installers Without Third-Party Tools
```cmd
iexpress
```
**What:** Wizard that packages files and a command into a self-extracting `.exe`. **Why:** Quick internal installers. **Danger:** 🟢 Safe

#### HT-03 · Driver Verifier
```cmd
verifier
verifier /reset
```
**What:** Stress-tests drivers to expose bugs with a BSOD instead of silent corruption. **Why:** Pinpoints the driver causing mystery crashes. Don't leave it on; know how to boot Safe Mode to `/reset`. **Danger:** 🔴 Advanced

#### HT-04 · WBEMTest: A WMI Query Playground
```cmd
wbemtest
```
**What:** GUI to connect to namespaces and run WQL. **Why:** Explore WMI classes, test queries before scripting. **Danger:** 🟡 Moderate

#### HT-05 · Projector Switch From the Command Line
```cmd
displayswitch /extend
displayswitch /clone
displayswitch /internal
displayswitch /external
```
**What:** Same as Win+P. **Why:** Script multi-monitor layouts or bind to a hotkey. **Danger:** 🟢 Safe

#### HT-06 · Securely Wipe Free Space
```cmd
cipher /w:D:\
```
**What:** Overwrites deallocated space (three passes). **Why:** Sanitize deleted data before selling a drive. Takes a long time. **Danger:** 🟡 Moderate

#### HT-07 · Bluetooth File Transfer Wizard
```cmd
fsquirt
```
**What:** Send or receive files over Bluetooth. **Why:** Faster than hunting through Settings. **Danger:** 🟢 Safe

#### HT-08 · Reset the Graphics Driver Without Rebooting
**Keys:** `Win` + `Ctrl` + `Shift` + `B`
**What:** Restarts the display stack (screen flashes, a beep sounds). **Why:** Recovers black screens, frozen displays and stuck GPU states while apps stay open. **Danger:** 🟢 Safe

#### HT-09 · Windows Memory Diagnostic
```cmd
mdsched
```
**What:** Schedules a RAM test at next reboot; results appear in Event Log (`MemoryDiagnostics-Results`). **Why:** Rules out bad RAM before blaming Windows. **Danger:** 🟡 Moderate

#### HT-10 · Show Ghost Devices in Device Manager
```cmd
set devmgr_show_nonpresent_devices=1
start devmgmt.msc
```
**What:** Then use *View → Show hidden devices*. **Why:** Uninstall stale NICs, COM ports and USB devices that cause conflicts. **Danger:** 🟡 Moderate

#### HT-11 · Copy Locked Files With esentutl
```cmd
esentutl /y "C:\path\locked.db" /d "D:\copy.db" /o
```
**What:** Copies files that are open and locked by another process. **Why:** Back up in-use databases and hives. Copy may not be transactionally consistent. **Danger:** 🟡 Moderate

#### HT-12 · List Minifilter Drivers
```cmd
fltmc
fltmc instances
```
**What:** Shows filesystem filter drivers (AV, backup, encryption) and their altitudes. **Why:** Prime suspects for weird I/O slowdowns and install failures. **Danger:** 🟢 Safe

#### HT-13 · TPM Details From the Console
```cmd
tpmtool getdeviceinformation
```
**What:** TPM version, manufacturer and health. **Why:** Check Windows 11 readiness quickly. **Danger:** 🟢 Safe

#### HT-14 · Map a Deep Path to a Drive Letter
```cmd
subst X: "C:\Projects\Client\Repo\src"
subst X: /d
```
**What:** Creates a virtual drive. **Why:** Sidestep long-path problems and shorten navigation. **Danger:** 🟢 Safe

#### HT-15 · Shell Folder Shortcuts
```cmd
explorer shell:AppsFolder
explorer shell:startup
explorer shell:sendto
explorer shell:Common Startup
```
**What:** Jumps to special folders by name. **Why:** `AppsFolder` lists every app, including Store apps, and lets you drag out shortcuts. **Danger:** 🟢 Safe

#### HT-16 · Native CAB Compression
```cmd
makecab /D CompressionType=LZX /D CompressionMemory=21 big.log big.cab
expand big.cab -F:* C:\out
```
**What:** Windows' own high-ratio compressor. **Why:** Compress logs on servers with no 7-Zip. **Danger:** 🟢 Safe

#### HT-17 · Printer Migration Tool
```cmd
printbrmui
```
**What:** Exports/imports printers, drivers and ports to a file. **Why:** Move a print setup between machines in minutes. **Danger:** 🟡 Moderate

#### HT-18 · View Stored Credentials
```cmd
cmdkey /list
```
**What:** Lists credentials saved by Windows (RDP, shares). **Why:** Find stale credentials causing account lockouts; remove with `cmdkey /delete:TARGET`. **Danger:** 🟢 Safe

#### HT-19 · Measure Clock Drift Against a Time Server
```cmd
w32tm /stripchart /computer:time.windows.com /samples:5 /dataonly
```
**What:** Shows offset to a time source. **Why:** Diagnose Kerberos failures (5-minute skew limit). **Danger:** 🟢 Safe

#### HT-20 · Write Your Own Event Log Entries
```cmd
eventcreate /t information /id 100 /l application /so MyScript /d "Backup completed"
```
**What:** Adds a custom event (Admin; ID 1–1000). **Why:** Scripts can log to the Event Log, which monitoring tools already read. **Danger:** 🟢 Safe

---

# 🤖 20 Automation Scripts

> Copy into `.ps1` files. Run elevated where needed, and test with `-WhatIf`/on a VM first.

#### AS-01 · Weekly Maintenance Orchestrator
```powershell
Start-Transcript "C:\Logs\maint-$(Get-Date -f yyyyMMdd).log"
DISM /Online /Cleanup-Image /RestoreHealth
sfc /scannow
Optimize-Volume -DriveLetter C -ReTrim
gci C:\Logs -File | ? LastWriteTime -lt (Get-Date).AddDays(-60) | Remove-Item
Stop-Transcript
```
**What:** Repairs the image, verifies system files, trims the SSD, and prunes old logs with a transcript. **Why:** Pair with [#79](#79-one-line-scheduled-task-as-system) for hands-free hygiene. **Danger:** 🟡 Moderate

#### AS-02 · Low-Disk Alert to a Teams/Slack Webhook
```powershell
$webhook = 'https://example.webhook.office.com/...'
Get-Volume | ? { $_.DriveLetter -and $_.SizeRemaining/$_.Size -lt 0.10 } | % {
  $msg = "$env:COMPUTERNAME $($_.DriveLetter): only $([math]::Round($_.SizeRemaining/1GB,1)) GB free"
  Invoke-RestMethod $webhook -Method Post -ContentType 'application/json' -Body (@{text=$msg} | ConvertTo-Json)
}
```
**What:** Posts a message when any volume drops under 10% free. **Why:** Cheap monitoring with no agent. Webhook payload format varies by service. **Danger:** 🟢 Safe

#### AS-03 · Age-Based Temp Cleaner
```powershell
foreach ($p in $env:TEMP, "$env:windir\Temp") {
  gci $p -Recurse -Force -EA 0 | ? LastWriteTime -lt (Get-Date).AddDays(-7) | Remove-Item -Recurse -Force -EA 0
}
```
**What:** Deletes temp files older than 7 days (locked files are skipped). **Why:** Safer than wiping everything. **Danger:** 🟡 Moderate

#### AS-04 · Zero-Touch Developer Box
```powershell
'Git.Git','Microsoft.PowerShell','Microsoft.VisualStudioCode','Microsoft.PowerToys','7zip.7zip' |
  % { winget install --id $_ -e --silent --accept-package-agreements --accept-source-agreements }
wsl --install -d Ubuntu
```
**What:** Installs your toolchain and WSL. **Why:** Fresh PC to productive in one script. For declarative setups, explore `winget configure -f dev.dsc.yaml`. **Danger:** 🟡 Moderate

#### AS-05 · Dated Backup With Retention
```powershell
$dst = "D:\Backups\$(Get-Date -f yyyy-MM-dd)"
robocopy "$HOME\Documents" $dst /E /MT:16 /R:1 /W:1 /NP | Out-Null
gci D:\Backups -Directory | sort CreationTime -Desc | select -Skip 14 | Remove-Item -Recurse -Force
```
**What:** Daily copy, keeps the newest 14. **Why:** Simple versioned backups. Verify `$dst` before enabling the prune line. **Danger:** 🟡 Moderate

#### AS-06 · Local Admin Audit Across Many PCs
```powershell
Invoke-Command -ComputerName (Get-Content .\computers.txt) -ErrorAction SilentlyContinue -ScriptBlock {
  Get-LocalGroupMember Administrators | select Name,ObjectClass
} | select PSComputerName,Name,ObjectClass | Export-Csv admins.csv -NoTypeInformation
```
**What:** Collects admin group membership fleet-wide via WinRM. **Why:** Instant privilege-creep report. **Danger:** 🟢 Safe

#### AS-07 · Daily Error Digest as HTML
```powershell
Get-WinEvent -FilterHashtable @{LogName='System','Application';Level=1,2;StartTime=(Get-Date).AddDays(-1)} -EA 0 |
  group ProviderName | sort Count -Desc | select Count,Name |
  ConvertTo-Html -Title 'Errors last 24h' | Out-File "$HOME\Desktop\errors.html"
```
**What:** Ranks noisy providers by error count. **Why:** Spot the one component that's failing 400 times a day. **Danger:** 🟢 Safe

#### AS-08 · React to New Files in a Folder
```powershell
$w = [IO.FileSystemWatcher]::new('C:\Inbox'); $w.EnableRaisingEvents = $true
Register-ObjectEvent $w Created -Action { "$(Get-Date -f s) $($Event.SourceEventArgs.FullPath)" | Add-Content C:\Logs\inbox.log }
```
**What:** Event-driven file watcher (lives as long as the session). **Why:** Build drop-folder workflows: auto-convert, auto-upload, auto-sort. **Danger:** 🟢 Safe

#### AS-09 · Service Watchdog With Event Log Entry
```powershell
'Spooler','W32Time' | % {
  if ((Get-Service $_).Status -ne 'Running') {
    Start-Service $_
    eventcreate /t warning /id 901 /l application /so Watchdog /d "Restarted $_" | Out-Null
  }
}
```
**What:** Restarts stopped services and records it. **Why:** Self-healing for flaky services; schedule every 5 minutes. **Danger:** 🟡 Moderate

#### AS-10 · Internet Latency Logger (PS7)
```powershell
while ($true) {
  $r = Test-Connection 1.1.1.1 -Count 1 -EA SilentlyContinue
  [pscustomobject]@{Time=(Get-Date -f s); LatencyMs=$r.Latency} | Export-Csv C:\Logs\net.csv -Append -NoTypeInformation
  Start-Sleep 60
}
```
**What:** One data point per minute in a CSV. **Why:** Evidence for your ISP; chart in Excel. (`ResponseTime` on PS 5.1.) **Danger:** 🟢 Safe

#### AS-11 · Defender Health Snapshot
```powershell
Get-MpComputerStatus | select AMServiceEnabled,RealTimeProtectionEnabled,IsTamperProtected,AntivirusSignatureLastUpdated,AntivirusSignatureAge,QuickScanEndTime
```
**What:** Protection status and signature freshness. **Why:** Compliance check you can run remotely. **Danger:** 🟢 Safe

#### AS-12 · Remove Consumer Bloat Packages
```powershell
foreach ($b in '*bingnews*','*zunevideo*','*solitaire*','*clipchamp*') {
  Get-AppxPackage -AllUsers $b | Remove-AppxPackage -AllUsers
}
```
**What:** Uninstalls selected Store apps. **Why:** Clean images for new PCs. Review each pattern; some apps are dependencies. **Danger:** 🟡 Moderate

#### AS-13 · Persistence Diff (Did Anything New Start Autorunning?)
```powershell
autorunsc64 -accepteula -a * -c -nobanner | ConvertFrom-Csv | Export-Clixml "$HOME\autoruns-$(Get-Date -f yyyyMMdd).xml"
Compare-Object (Import-Clixml .\autoruns-yesterday.xml) (Import-Clixml .\autoruns-today.xml) -Property Entry,'Image Path'
```
**What:** Baselines all auto-start locations and compares days. **Why:** Lightweight change detection for malware or bad installers. **Danger:** 🟢 Safe

#### AS-14 · Restore Point Before Risky Changes
```powershell
Enable-ComputerRestore -Drive 'C:\'
Checkpoint-Computer -Description 'Before tweaks' -RestorePointType MODIFY_SETTINGS
```
**What:** Creates a System Restore point (Windows throttles to one per 24h by default). **Why:** Put at the top of any tweak script. **Danger:** 🟢 Safe

#### AS-15 · WSL Distro Bootstrap and Backup
```powershell
wsl --install -d Ubuntu
wsl -d Ubuntu -u root -- bash -c "apt update && apt -y upgrade"
wsl --export Ubuntu D:\Backups\ubuntu.tar
```
**What:** Installs, patches and snapshots a Linux environment. **Why:** Reproducible dev environments; `wsl --import` restores anywhere. **Danger:** 🟡 Moderate

#### AS-16 · Patch-Day Checkpoints for All Running VMs
```powershell
Get-VM | ? State -eq Running | % { Checkpoint-VM $_ -SnapshotName "patch-$(Get-Date -f yyyyMMdd)" }
Get-VM | Get-VMSnapshot | ? CreationTime -lt (Get-Date).AddDays(-7) | Remove-VMSnapshot
```
**What:** Safety snapshot before updates, then prune week-old ones. **Why:** Instant rollback for lab/test VMs. Not for domain controllers. **Danger:** 🟡 Moderate

#### AS-17 · Resumable, Low-Priority Downloads With BITS
```powershell
Start-BitsTransfer -Source 'https://example.com/big.iso' -Destination C:\dl\big.iso -Priority Low -DisplayName 'ISO'
```
**What:** Background transfer that survives reboots and network drops. **Why:** Big downloads that don't hog bandwidth; add `-Asynchronous` for fire-and-forget. **Danger:** 🟢 Safe

#### AS-18 · Disk-Space Trend CSV
```powershell
Get-Volume | ? DriveLetter | select @{n='Date';e={Get-Date -f s}},DriveLetter,@{n='FreeGB';e={[math]::Round($_.SizeRemaining/1GB,1)}} | Export-Csv C:\Logs\space.csv -Append -NoTypeInformation
```
**What:** Appends daily free space. **Why:** Graph consumption and predict when you'll run out. **Danger:** 🟢 Safe

#### AS-19 · Unattended Windows Updates via PSWindowsUpdate
```powershell
Install-Module PSWindowsUpdate -Scope AllUsers -Force
Get-WindowsUpdate -MicrosoftUpdate -Install -AcceptAll -IgnoreReboot
```
**What:** Installs updates (including Microsoft products) from the shell. **Why:** Patch servers and Core installs without the GUI. **Danger:** 🟡 Moderate

#### AS-20 · Log Off Disconnected RDP Sessions
```powershell
quser | ? { $_ -match 'Disc' } | % { ($_ -split '\s+')[2] } | % { logoff $_ }
```
**What:** Frees sessions left behind by closed RDP windows. **Why:** Reclaims licenses and RAM on shared servers. Forces logoff, so unsaved work is lost. Test the ID parsing first by dropping the last stage. **Danger:** 🟡 Moderate

---

# 🎉 10 Fun (Safe) Commands

> Some use third-party web services, so they need internet and may occasionally be offline.

#### FUN-01 · ASCII Weather Forecast
```powershell
curl.exe wttr.in
```
**What:** Draws a colorful text forecast for your IP-detected location. **Why:** Terminal flex. **Danger:** 🟢 Safe

#### FUN-02 · Dancing Parrot in Your Terminal
```powershell
curl.exe parrot.live
```
**What:** Streams an ASCII-art animation. **Why:** Perfect Windows Terminal demo. Ctrl+C to exit. **Danger:** 🟢 Safe

#### FUN-03 · Matrix Mode
```cmd
cmd /k "color 0a & tree C:\Windows /f"
```
**What:** Green-on-black falling file tree. **Why:** Looks like a movie hacker scene. **Danger:** 🟢 Safe

#### FUN-04 · Make Windows Talk
```powershell
Add-Type -AssemblyName System.Speech; (New-Object System.Speech.Synthesis.SpeechSynthesizer).Speak("Backup complete. Have a nice day.")
```
**What:** Text-to-speech. **Why:** Audible alerts at the end of long scripts. **Danger:** 🟢 Safe

#### FUN-05 · Play a Tune With the PC Speaker API
```powershell
@(392,392,392,311,466,392,311,466,392) | % { [console]::Beep($_,300) }
```
**What:** Plays a melody (frequency in Hz). **Why:** Fun, and a handy "job finished" beep. **Danger:** 🟢 Safe

#### FUN-06 · Rock, Paper, Scissors
```powershell
Get-Random -InputObject 'Rock','Paper','Scissors'
```
**What:** Random pick. **Why:** `Get-Random -InputObject` also picks winners from lists. **Danger:** 🟢 Safe

#### FUN-07 · "Hacker Typer" Screen
```powershell
1..60 | % { -join (1..100 | % { [char](Get-Random -Min 33 -Max 127) }); Start-Sleep -Milliseconds 40 }
```
**What:** Waterfalls of random characters. **Why:** Pointless and delightful. **Danger:** 🟢 Safe

#### FUN-08 · 256-Color Terminal Test (PS7)
```powershell
0..255 | % { Write-Host "$([char]27)[48;5;${_}m  $([char]27)[0m" -NoNewline; if (($_+1) % 16 -eq 0) { '' } }
```
**What:** Draws the full ANSI palette. **Why:** Confirms your terminal and theme render colors correctly. **Danger:** 🟢 Safe

#### FUN-09 · Rocket Countdown
```powershell
10..1 | % { Write-Host $_ -ForegroundColor Yellow; Start-Sleep 1 }; Write-Host '🚀 Liftoff!' -ForegroundColor Green
```
**What:** A dramatic delay. **Why:** Good template for "wait N seconds" prompts. **Danger:** 🟢 Safe

#### FUN-10 · Jump to Any Settings Page by URI
```cmd
start ms-settings:easeofaccess-colorfilter
start ms-settings:about
start ms-screenclip:
```
**What:** Windows registers `ms-settings:`-style URIs. **Why:** Deep links for scripts and shortcuts. Search "ms-settings URI list" for hundreds more. **Danger:** 🟢 Safe

---

# 🏢 10 Commands IT Professionals Live By

#### IT-01 · Group Policy Result as an HTML Report
```cmd
gpresult /h "%USERPROFILE%\Desktop\gp.html" /f
gpresult /scope computer /h C:\gp-computer.html /f
```
**What:** Shows which GPOs applied, which were filtered, and the winning setting (Admin for computer scope). **Why:** First stop for "why doesn't this policy apply?". **Danger:** 🟢 Safe

#### IT-02 · Entra ID / Hybrid Join Status
```cmd
dsregcmd /status
```
**What:** Join state, PRT status, tenant info, and SSO diagnostics. **Why:** Solves most "Conditional Access/SSO is broken" cases. **Danger:** 🟢 Safe

#### IT-03 · Secure Channel and DC Locator
```cmd
nltest /sc_verify:corp.contoso.com
nltest /dsgetdc:corp.contoso.com
```
**What:** Validates the machine's trust with the domain and shows which DC it finds. **Why:** Fixes "trust relationship failed" without rejoining. Replace the domain with your own. **Danger:** 🟢 Safe

#### IT-04 · Kerberos Ticket Inspection
```cmd
klist
klist purge
```
**What:** Lists cached tickets; `purge` clears them. **Why:** Reveals stale group membership and SPN problems; purge to pick up new group memberships without logging off. **Danger:** 🟡 Moderate

#### IT-05 · BitLocker Status and Protectors
```cmd
manage-bde -status
manage-bde -protectors -get C:
```
**What:** Encryption state, method and key protectors (Admin). **Why:** Audit and back up recovery info. **Treat output as sensitive.** **Danger:** 🟡 Moderate

#### IT-06 · Run a Command on a Whole Fleet
```powershell
Invoke-Command -ComputerName (Get-Content pcs.txt) -ScriptBlock { Get-CimInstance Win32_OperatingSystem | select CSName,Version,LastBootUpTime } -ThrottleLimit 32 | ft -Auto
```
**What:** Parallel remoting over WinRM. **Why:** The foundation of scalable Windows administration. Requires `Enable-PSRemoting` and permissions. **Danger:** 🟡 Moderate

#### IT-07 · Domain Controller Health Check
```cmd
dcdiag /q
```
**What:** Runs diagnostic tests and prints **only failures** (`/q`). **Why:** Quiet means healthy. Run on a DC. **Danger:** 🟢 Safe

#### IT-08 · AD Replication Summary
```cmd
repadmin /replsummary
repadmin /showrepl
```
**What:** Replication latency and error counts per DC. **Why:** Detect lingering replication failures before users do. **Danger:** 🟢 Safe

#### IT-09 · Find Stale Computer Accounts (RSAT)
```powershell
Get-ADComputer -Filter * -Properties LastLogonDate | ? LastLogonDate -lt (Get-Date).AddDays(-90) | sort LastLogonDate | select Name,LastLogonDate,DistinguishedName
```
**What:** Machines that haven't logged on in 90 days. **Why:** AD hygiene and license reclamation. Review before disabling or deleting. **Danger:** 🟢 Safe

#### IT-10 · Certificate Chain and Revocation Test
```cmd
certutil -urlfetch -verify C:\certs\server.cer
certutil -store -user my
```
**What:** Validates the chain, fetches CRL/OCSP endpoints and reports failures. **Why:** Diagnoses "certificate not trusted/revoked" problems and unreachable CDPs. **Danger:** 🟢 Safe

---

## 🤝 Contributing

1. Fork, then add your entry using the format below.
2. Test on a clean VM and state minimum Windows version / elevation.
3. Include a *what*, *why*, and honest **danger level**.
4. Open a PR. Prefer surprising and useful over obvious.

````markdown
#### N. Title
```powershell
command
```
**What:** … **Why:** … **Danger:** 🟢 Safe | 🟡 Moderate | 🔴 Advanced
````

## 📜 License

MIT. Use at your own risk. No warranty. Always back up before changing system configuration.
