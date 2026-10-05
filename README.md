# Windows 10 — Customized Build Guide

> **Personal Windows 10 22H2 image-customization notes**  
> Offline servicing → Audit Mode customization → final cleanup → WIM/ESD/ISO

![Windows 10](https://img.shields.io/badge/Windows%2010-22H2-blue?logo=windows)
![Architecture](https://img.shields.io/badge/Architecture-x64-lightgrey)
![Build](https://img.shields.io/badge/Reference%20build-2026-informational)

This repository records a repeatable process for building a **lean, customized Windows 10 22H2 x64 reference image**. The workflow is intentionally split into offline servicing, online Audit Mode customization, and final offline cleanup so that each stage can be inspected independently.

### 🎯 Design goals

- Keep the Windows 10 base system lean without turning the process into a blind debloat script.
- Add **Simplified Chinese fonts and Microsoft Pinyin** while retaining English as the Windows display language.
- Remove selected optional components, provisioned AppX packages, services, scheduled tasks, and bundled payloads according to personal requirements.
- Produce both a deployment-friendly `install.wim` and a compressed `install.esd` for installation media.
- Refresh the reference image **annually**; the current build is the **2026** image.

### 🧭 Workflow

```text
Source ISO / FOD / drivers
          │
          ▼
Stage 1 — Offline customization
          │
          ├─ WinRE fonts
          ├─ CJK fonts + Pinyin
          ├─ drivers
          ├─ curated feature/capability/AppX/package changes
          ├─ payload cleanup
          └─ Audit-mode unattend
          │
          ▼
Stage WIM → Audit Mode VHDX
          │
          ▼
Stage 2 — Online customization
          │
          ├─ Reserved Storage
          ├─ Edge / OneDrive cleanup
          ├─ privacy policies
          ├─ services / scheduled tasks
          ├─ CompactOS
          └─ Sysprep
          │
          ▼
Stage 3 — Final offline cleanup
          │
          ├─ component-store cleanup
          ├─ build-artifact cleanup
          ├─ boot optimization
          └─ capture final WIM
          │
          ├──────────────┐
          ▼              ▼
       install.wim   install.esd
          │              │
          └───────┬──────┘
                  ▼
             Bootable ISO
```

> **Note:** This document intentionally favors transparency and manual review over full automation. The resulting image should be tested in a VM before use on physical hardware.

## 📑 Contents

1. [Offline customization](#stage-1--offline-customization)
2. [Audit Mode customization](#stage-2--online-customization-in-audit-mode)
3. [Final cleanup and installation media](#stage-3--final-cleanup-and-installation-media)
4. [Post-installation](#post-installation)
5. [Final validation](#-final-validation-checklist)
6. [Annual image refresh](#-annual-image-refresh)


## ⚠️ Disclaimer

This repository documents a **personal Windows 10 optimization and image-customization workflow**. It is a set of notes and reminders, **not a fully automated script**.

> **Do not paste-and-run the commands blindly.**  
> Most steps require manual inspection and judgment before proceeding.

A few conventions used throughout this document:

- `*` in a path is **not** intended to indicate a shell wildcard command. It means "inspect matching files/items under this location" before deciding what to remove.
- Debloat operations are intentionally **manual and curated**. Review each item and understand its purpose before removing or disabling it.
- This workflow is designed for a **non-production, personal/reference image**. Validate the resulting image in a VM before deploying it to real hardware.
- Commands may need to be adjusted for your edition, source media, architecture, mounted drive letters, and servicing baseline.

**Use at your own risk. No warranty is provided.**

## 📌 Build Variables

Pre-defined working directories:

```
set "WORKDIR=D:\win10\work"
set "MOUNTDIR=D:\win10\mount"
set "RE_MOUNTDIR=D:\win10\remount"
set "OUTPUT=%WORKDIR%\output"
set "IMAGEYEAR=2026"
```

## 📦 Source Media & Prerequisites
**Windows 10 22H2 (19045) Feature Update ISO**  
[UUP dump — Windows 10 22H2](https://uupdump.net/known.php?q=category:w10-22h2)

**Windows 10 Language Pack / Features on Demand media**  
[Microsoft — Language packs and Features on Demand](https://learn.microsoft.com/en-us/azure/virtual-desktop/language-packs)

**Intel Wi-Fi AX210 drivers**  
[Intel — Wi-Fi 6E AX210 downloads](https://www.intel.com/content/www/us/en/products/sku/204836/intel-wifi-6e-ax210-gig/downloads.html)

| Item | Purpose |
|---|---|
| Windows 10 22H2 ISO | Base installation source |
| Windows 10 Language Pack / FOD media | CJK language capabilities and fonts |
| Intel AX200/AX210 driver package | Offline network driver integration |

> Keep the Windows source, FOD/language media, and drivers matched to the intended architecture and servicing baseline.

# Stage 1 — Offline Customization

## 1. Mount the Installation WIM

Identify the target edition/index before servicing the image.

### List available indexes
```
dism /Get-WimInfo /WimFile:%WORKDIR%\install.wim
```

### Check WIM integrity
```
dism /Check-Integrity /WimFile:%WORKDIR%\install.wim
```

### Mount the source WIM
```
dism /Mount-Wim /WimFile:%WORKDIR%\install.wim /Index:1 /MountDir:%MOUNTDIR%
dism /Get-MountedWimInfo
```

## 2. Rebuild WinRE
Copy `%MOUNTDIR%\Windows\System32\Recovery\winre.wim` to `%WORKDIR%`.

### Mount WinRE
```
dism /Get-WimInfo /WimFile:%WORKDIR%\winre.wim
dism /Mount-Wim /WimFile:%WORKDIR%\winre.wim /Index:1 /MountDir:%RE_MOUNTDIR%
dism /Get-MountedWimInfo
```

### Add Simplified Chinese font support
```
dism /Image:%RE_MOUNTDIR% /Add-Package /PackagePath:%WORKDIR%\re\WinPE-FontSupport-ZH-CN.cab /LimitAccess
dism /Image:%RE_MOUNTDIR% /Get-Packages | findstr /i "FontSupport"
```

### Cleanup, commit, and export
```
dism /Image:%RE_MOUNTDIR% /Cleanup-Image /StartComponentCleanup /Resetbase
dism /Unmount-Image /MountDir:%RE_MOUNTDIR% /Commit
dism /Export-Image /SourceImageFile:%WORKDIR%\winre.wim /SourceIndex:1 /DestinationImageFile:%WORKDIR%\winre_rebuild.wim /Compress:max /CheckIntegrity
```

Copy `winre_rebuild.wim` to `%MOUNTDIR%\Windows\System32\Recovery\winre.wim`.

## 3. Integrate CJK Fonts and Microsoft Pinyin IME
Add **Chinese basic typing support** and **Simplified Chinese/Han fonts**. This does **not** install the Chinese display language, OCR, handwriting, or text-to-speech components.
```
dism /Image:%MOUNTDIR% /Add-Capability /CapabilityName:Language.Basic~~~zh-CN~0.0.1.0 /Source:%WORKDIR%\lang /LimitAccess
dism /Image:%MOUNTDIR% /Add-Capability /CapabilityName:Language.Fonts.Hans~~~und-HANS~0.0.1.0 /Source:%WORKDIR%\lang /LimitAccess
```

### Verify language capabilities
```
dism /Image:"%MOUNTDIR%" /Get-Capabilities | findstr /i "Language"
dism /Image:"%MOUNTDIR%" /Get-CapabilityInfo /CapabilityName:Language.Basic~~~zh-CN~0.0.1.0
```

## 4. Integrate Drivers
### Add Intel AX200/AX210 drivers
```
dism /Image:%MOUNTDIR% /Add-Driver /Driver:%WORKDIR%\drivers\Netwtw08.INF
```

### Verify installed drivers
```
dism /Image:%MOUNTDIR% /Get-Drivers
```

## 5. Disable Optional Windows Features

Export the current feature inventory, review it manually, and create a curated list of features to disable.

### Export the feature inventory
```
dism /Image:%MOUNTDIR% /Get-Features /Format:Table > %WORKDIR%\feature_remove.txt
```

> **Manual step:** Curate the disable list before continuing. Keep **one feature name per line**.

### Disable selected features
```
for /f "usebackq delims=" %%P in ("%WORKDIR%\feature_remove.txt") do (dism /Image:%MOUNTDIR% /Disable-Feature /FeatureName:%%P /Remove)
```

### Verify feature state
```
dism /Image:%MOUNTDIR% /Get-Features /Format:Table
```

## 6. Remove Optional Capabilities

Export the current capability inventory, review it manually, and create a curated removal list.

### Export the capability inventory
```
dism /Image:%MOUNTDIR% /Get-Capabilities /Format:Table > %WORKDIR%\cap_remove.txt
```

> **Manual step:** Curate the removal list before continuing. Keep **one capability name per line**.

### Remove selected capabilities
```
for /f "usebackq delims=" %%P in ("%WORKDIR%\cap_remove.txt") do (dism /Image:%MOUNTDIR% /Remove-Capability /CapabilityName:%%P)
```

### Verify capability state
```
dism /Image:%MOUNTDIR% /Get-Capabilities /Format:Table
```

## 7. Remove Provisioned AppX Packages

Export the current provisioned-AppX inventory, review it manually, and create a curated removal list.

### Export the AppX inventory
```
dism /Image:%MOUNTDIR% /Get-ProvisionedAppxPackages > %WORKDIR%\appx_remove.txt
```

> **Manual step:** Curate the removal list before continuing. Keep **one AppX package name per line**.

### Remove selected AppX packages
```
for /f "usebackq delims=" %%P in ("%WORKDIR%\appx_remove.txt") do (dism /Image:%MOUNTDIR% /Remove-ProvisionedAppxPackage /PackageName:%%P)
```

### Verify provisioned AppX packages
```
dism /Image:%MOUNTDIR% /Get-ProvisionedAppxPackages
```

### Optimize provisioned AppX packages
```
dism /Image:%MOUNTDIR% /Optimize-ProvisionedAppxPackages
```

## 8. Commit Changes and Remount
```
dism /Unmount-Image /MountDir:%MOUNTDIR% /Commit
dism /Mount-Wim /WimFile:%WORKDIR%\install.wim /Index:1 /MountDir:%MOUNTDIR%
dism /Get-MountedWimInfo
```

## 9. Remove Servicing Packages

Export the visible package inventory and remove only packages that you have explicitly reviewed.

### Export the package inventory
```
dism /Image:%MOUNTDIR% /Get-Packages /Format:Table > %WORKDIR%\package_remove.txt
```

> **Manual step:** Curate the removal list before continuing. Keep **one package name per line**.

### Remove selected packages
```
for /f "usebackq delims=" %%P in ("%WORKDIR%\package_remove.txt") do (dism /Image:%MOUNTDIR% /Remove-Package /PackageName:%%P)
```

### Verify installed packages
```
dism /Image:%MOUNTDIR% /Get-Packages /Format:Table
```

## 10. Remove Other Payloads
### Microsoft Edge payloads

> **WebView2 is intentionally not removed.** Edge/WebView-related components should be evaluated separately from the browser itself.
```
%MOUNTDIR%\ProgramData\Microsoft\EdgeUpdate
%MOUNTDIR%\WindowsApps\Microsoft.MicrosoftEdge*
%MOUNTDIR%\Program Files\WindowsApps\Microsoft.MicrosoftEdge*
%MOUNTDIR%\Program Files (x86)\Microsoft\Edge\
%MOUNTDIR%\Program Files (x86)\Microsoft\EdgeUpdate\
%MOUNTDIR%\Program Files (x86)\Microsoft\Temp\
%MOUNTDIR%\Windows\SystemApps\Microsoft.MicrosoftEdge*
%MOUNTDIR%\Windows\System32\MicrosoftEdge*
```

### OneDrive payloads
```
%MOUNTDIR%\Windows\System32\OneDriveSetup.exe
%MOUNTDIR%\Windows\SysWOW64\OneDriveSetup.exe
```

### Wallpapers, lock screen, and themes
```
%MOUNTDIR%\Windows\Web\Wallpaper
%MOUNTDIR%\Windows\Web\Screen
%MOUNTDIR%\Windows\Resources\Themes
```

### Sample media and Retail Demo content
```
%MOUNTDIR%\Users\Public\Pictures
%MOUNTDIR%\Users\Public\Music
%MOUNTDIR%\Users\Public\Videos
%MOUNTDIR%\Users\Public\Documents
%MOUNTDIR%\Windows\RetailDemo
```

## 11. Inject Audit-Mode Unattend
Create `Unattend.xml in `%MOUNTDIR%\Windows\Panther\Unattend`:
```
<?xml version="1.0" encoding="utf-8"?>
<unattend xmlns="urn:schemas-microsoft-com:unattend">
    <settings pass="oobeSystem">
        <component name="Microsoft-Windows-Deployment"
                   processorArchitecture="amd64"
                   publicKeyToken="31bf3856ad364e35"
                   language="neutral"
                   versionScope="nonSxS">
            <Reseal>
                <Mode>Audit</Mode>
            </Reseal>
        </component>
    </settings>
</unattend>
```

## 12. Commit and Export the Stage WIM
### Commit and unmount
```
dism /Unmount-Image /MountDir:%MOUNTDIR% /Commit
```

### Export the stage WIM
```
dism /Get-WimInfo /WimFile:%WORKDIR%\install.wim
dism /Export-Image /SourceImageFile:%WORKDIR%\install.wim /SourceIndex:1 /DestinationImageFile:%WORKDIR%\install_stage.wim /Compress:max /CheckIntegrity
dism /Get-WimInfo /WimFile:%WORKDIR%\install_stage.wim
```


# Stage 2 — Online Customization in Audit Mode

## 13. Prepare the VM and Boot into Audit Mode
Create a VM with a **30 GB VHDX**. Use DiskPart to create an EFI System Partition and a Windows/system partition, then assign the Windows partition to `V:`.

### Apply the stage WIM to the VHDX
```
dism /Apply-Image /ImageFile:%WORKDIR%\install_stage.wim /Index:1 /ApplyDir:V:\
dism /Image:V: /Optimize-Image /Boot
```

### Detach and boot into WinRE

Boot the VM from the original installation media into WinRE and assign the EFI partition to `S:`.

Make the VHDX bootable; assume `C:` is the Windows/system partition.
```
bcdboot C:\Windows /s S: /f UEFI
```

Reboot the VM. Windows should enter **Audit Mode** and complete any pending CBS servicing operations.

## 14. Online Tweaks
### Disable Reserved Storage
```
dism /Online /Get-ReservedStorageState
dism /Online /Set-ReservedStorageState /State:Disabled
dism /Online /Get-ReservedStorageState
```

### Edge and OneDrive cleanup

Edge and OneDrive are intentionally removed from the base image. **Edge WebView is not removed.** If Edge, OneDrive, or WebView2 is later required, reinstall the corresponding component manually.

- Identify the relevant services from `sc query | findstr /i "Edge"` and `sc query type= all | findstr /i "OneDrive"` manually, then stop/delete only the services selected for removal. These registrations may return when Edge or OneDrive is installed again.

- Review and delete the relevant scheduled tasks using `schtasks /query /fo LIST | findstr /i "Edge"` and `schtasks /query /fo LIST | findstr /i "OneDrive"`.

- Clean the following registry locations as applicable:
```
HKLM\SOFTWARE\Microsoft\Edge
HKLM\SOFTWARE\WOW6432Node\Microsoft\Edge
HKLM\SOFTWARE\Microsoft\EdgeUpdate
HKLM\SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate
HKLM\SOFTWARE\Policies\Microsoft\Edge
HKLM\SOFTWARE\Policies\Microsoft\EdgeUpdate

HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\Microsoft Edge
HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\Microsoft Edge

HKCR\Applications\msedge.exe
HKCR\msedge
HKCR\microsoft-edge
HKCR\microsoft-edge-holographic

HKLM\SOFTWARE\Microsoft\OneDrive
HKLM\SOFTWARE\WOW6432Node\Microsoft\OneDrive

HKLM\SOFTWARE\Policies\Microsoft\Windows\OneDrive

HKCU\Software\Microsoft\OneDrive
HKCU\Environment\OneDrive
HKCR\CLSID\{018D5C66-4533-4307-9B53-224DE2ED1FE6}
```

### Privacy and telemetry configuration

#### Minimum telemetry and diagnostic data
```
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v AllowTelemetry /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v MaxTelemetryAllowed /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v AllowDeviceNameInTelemetry /t REG_DWORD /d 0 /f
```

#### Disable Advertising ID
```
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\AdvertisingInfo" /v DisabledByGroupPolicy /t REG_DWORD /d 1 /f
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\AdvertisingInfo" /v Enabled /t REG_DWORD /d 0 /f
```

#### Block consumer features and sponsored apps
```
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\CloudContent" /v DisableWindowsConsumerFeatures /t REG_DWORD /d 1 /f
```

#### Disable Cortana and web search
```
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search" /v AllowCortana /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search" /v DisableWebSearch /t REG_DWORD /d 1 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search" /v ConnectedSearchUseWeb /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search" /v AllowSearchToUseLocation /t REG_DWORD /d 0 /f
```

#### Disable News and Interests
```
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Feeds" /v EnableFeeds /t REG_DWORD /d 0 /f
```

#### Disable feedback notifications
```
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection" /v NumberOfSIUFInPeriod /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection" /v PeriodInNanoSeconds /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v DoNotShowFeedbackNotifications /t REG_DWORD /d 1 /f
```

#### Disable Activity History
```
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\System" /v EnableActivityFeed /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\System" /v PublishUserActivities /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\System" /v UploadUserActivities /t REG_DWORD /d 0 /f
```

#### Disable Location Services
```
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\LocationAndSensors" /v DisableLocation /t REG_DWORD /d 1 /f
```

### Disable selected services

The following list is based on a **manual review / BlackViper reference**. Keep only services that match your intended device scenarios. Items marked `**` are typically associated with optional/server-style functionality.
```
ActiveX Installer (AxInstSV)
Application Layer Gateway Service
Application Management
BranchCache
CaptureService_?????
Certificate Propagation
Connected User Experiences and Telemetry
Distributed Link Tracking Client
Downloaded Maps Manager
Fax **
Geolocation Service
Infrared monitor service
Interactive Services Detection
LPD Service **
Message Queuing **
Message Queuing Triggers **
Microsoft FTP Service **
Microsoft iSCSI Initiator Service
Microsoft Keyboard Filter **
Microsoft Windows SMS Router Service
MultiPoint Repair Service **
MultiPoint Service **
Net.Msmq Listener Adapter **
Net.Pipe Listener Adapter **
Net.Tcp Listener Adapter **
Net.Tcp Port Sharing Service **
Offline Files
Parental Controls
Payments and NFC/SE Manager
Performance Logs & Alerts
Phone Service
Program Compatibility Assistant Service
Retail Demo Service
RIP Listener **
Shell Hardware Detection
Simple TCP/IP Services **
Smart Card Device Enumeration Service
Smart Card Removal Policy
SNMP Service **
SNMP Trap
Spatial Data Service
Telephony
Touch Keyboard and Handwriting Panel Service
WebClient
Windows Connect Now - Config Registrar
Windows Insider Service
Windows Media Player Network Sharing Service **
Windows Mobile Hotspot Service
Windows Remote Management (WS-Management)
Work Folders **
WWAN AutoConfig
```

### Disable selected scheduled tasks
```
\Microsoft\Windows\Application Experience\MareBackup
\Microsoft\Windows\Application Experience\Microsoft Compatibility Appraiser
\Microsoft\Windows\Application Experience\PcaPatchDbTask
\Microsoft\Windows\Application Experience\PcaWallpaperAppDetect
\Microsoft\Windows\AppListBackup\Backup
\Microsoft\Windows\AppListBackup\BackupNonMaintenance
\Microsoft\Windows\Customer Experience Improvement Program\Consolidator
\Microsoft\Windows\Customer Experience Improvement Program\UsbCeip
\Microsoft\Windows\Diagnosis\RecommendedTroubleshootingScanner
\Microsoft\Windows\Diagnosis\Scheduled
\Microsoft\Windows\EDP\EDP App Launch Task
\Microsoft\Windows\EDP\EDP Auth Task
\Microsoft\Windows\EDP\EDP Inaccessible Credentials Task
\Microsoft\Windows\EDP\StorageCardEncryption Task
\Microsoft\Windows\Flighting\FeatureConfig\UsageDataFlushing
\Microsoft\Windows\Flighting\FeatureConfig\UsageDataReporting
\Microsoft\Windows\Location\Notifications
\Microsoft\Windows\Location\WindowsActionDialog
\Microsoft\Windows\Maintenance\WinSAT
\Microsoft\Windows\Maps\MapsToastTask
\Microsoft\Windows\Maps\MapsUpdateTask
\Microsoft\Windows\Mobile Broadband Accounts\MNO Metadata Parser
\Microsoft\Windows\NetTrace\GatherNetworkInfo
\Microsoft\Windows\PushToInstall\LoginCheck
\Microsoft\Windows\PushToInstall\Registration
\Microsoft\Windows\RemoteAssistance\RemoteAssistanceTask
\Microsoft\Windows\SettingSync\BackgroundUploadTask
\Microsoft\Windows\SettingSync\NetworkStateChangeTask
\Microsoft\Windows\Speech\SpeechModelDownloadTask
\Microsoft\Windows\Windows Error Reporting\QueueReporting
\Microsoft\Windows\WwanSvc\NotificationTask
\Microsoft\Windows\WwanSvc\OobeDiscovery
```

## 15. CompactOS
```
compact /CompactOs:always
```

## 16. Sysprep
> Run Sysprep only after the image has reached its intended final state.
```
%WINDIR%\System32\Sysprep\Sysprep.exe /generalize /oobe /shutdown
```

# Stage 3 — Final Cleanup and Installation Media

## 17. Attach the VHDX
Assign SYSTEM partition to V:

## 18. Component Store Cleanup
### Check component-store health and size
```
dism /Image:V: /Cleanup-Image /ScanHealth
dism /Image:V: /Cleanup-Image /AnalyzeComponentStore
```

### Clean superseded components
```
dism /Image:V: /Cleanup-Image /StartComponentCleanup /Resetbase
```

### Clean and hide superseded service-pack files
```
DISM /Image:V: /Cleanup-Image /SPSuperseded /HideSP
```

### Re-check component-store size
```
dism /Image:V: /Cleanup-Image /AnalyzeComponentStore
```

## 19. NTFS Compression Targets
```
V:\Windows\WinSxS
V:\Windows\servicing\LCU
V:\Windows\System32\DriverStore\FileRepository
V:\Windows\Installer
V:\Windows\SoftwareDistribution
```

## 20. Remove Remaining Build Artifacts
> **Important:** Remove the Audit-mode unattend file before capturing the final image.
```
V:\Windows\Panther\Unattend\Unattend.xml
V:\Windows\Panther\*.log
```

Remove WinSxS backup `V:\Windows\WinSxS\Backup`


### Remove runtime/build-cache files
```
V:\Windows\Temp\*
V:\Windows\Logs\*
V:\Windows\SoftwareDistribution\Download\*
V:\Windows\SoftwareDistribution\DataStore\*
V:\Windows\Downloaded Program Files\*
%TEMP%\*
``` 

## 21. Capture the Final WIM

> **Order matters:** `/Optimize-Image /Boot` is intentionally the last DISM image operation before capture.

### Optimize the image for boot performance
```
dism /Image:V: /Optimize-Image /Boot
```

### Capture immediately after optimization
```
Dism /Capture-Image /ImageFile:%WORKDIR%\install_stage2.wim /CaptureDir:V: /Name:"Windows 10 Pro" /Description:"Windows 10 Pro 22H2 customized image - %IMAGEYEAR%" /Compress:max /CheckIntegrity
```

### Mount the captured WIM for final metadata/export preparation
```
dism /Mount-Wim /WimFile:%WORKDIR%\install_stage2.wim /Index:1 /MountDir:%MOUNTDIR%
```

### Commit and unmount
```
dism /Unmount-Image /MountDir:%MOUNTDIR% /Commit
```

### Export the final `install.wim`
```
dism /Export-Image /SourceImageFile:%WORKDIR%\install_stage2.wim /SourceIndex:1 /DestinationImageFile:%OUTPUT%\install_oscurrent%IMAGEYEAR%.wim /Compress:max /CheckIntegrity
dism /Get-WimInfo /WimFile:%OUTPUT%\install_oscurrent%IMAGEYEAR%.wim
```

## 22. Build the Installation ISO
### Export `install.esd`

> The ESD is intended for installation media; the maximum-compression WIM is retained separately for servicing/deployment workflows.

```
dism /Export-Image /SourceImageFile:%WORKDIR%\install_stage2.wim /SourceIndex:1 /DestinationImageFile:%WORKDIR%\install.esd /Compress:recovery /CheckIntegrity
```

Copy `install.esd` to `dvd\sources` and replace the existing `install.wim`.

### Create the ISO
```
oscdimg -m -o -u2 -udfver102 -b%WORKDIR%\dvd\boot\etfsboot.com -e -bootdata:2#p0,e,b%WORKDIR%\dvd\boot\etfsboot.com#pEF,e,b%WORKDIR%\dvd\efi\microsoft\boot\efisys.bin %WORKDIR%\dvd\ %OUTPUT%\en_win10_22H2_custom_%IMAGEYEAR%.iso
```

Detach the VHDX and clean the working directory.

# Post-Installation

## 23. Deploy and Register WinRE
```
xcopy /h C:\Windows\System32\Recovery\Winre.wim C:\Recovery\WindowsRE
C:\Windows\System32\Reagentc /setreimage /path C:\Recovery\WindowsRE /target C:\Windows
reagentc /info
```

## 24. Windows Update Policy
Apply the Windows Update policy **after all intended servicing and validation are complete**.
```
Computer Configuration\Administrative Templates\Windows Components\Windows Update\Configure Automatic Updates
Computer Configuration\Administrative Templates\Windows Components\Windows Update\Manage end user experience
```

### Optional service-level restrictions
```
Windows Update
Background Intelligent Transfer Service
Update Orchestrator Service for Windows Update
Windows Update Medic Service (may reject as is protected)
```

---

## ✅ Final Validation Checklist

Before publishing or deploying a yearly image:

- [ ] Confirm the intended Windows 10 22H2 edition/index.
- [ ] Verify WIM integrity before and after major export operations.
- [ ] Confirm Chinese input and Microsoft Pinyin work while the UI remains English.
- [ ] Confirm WinRE boots and can display Simplified Chinese text.
- [ ] Verify networking and integrated Intel Wi-Fi drivers.
- [ ] Verify removed AppX packages, capabilities, features, and services against the intended hardware/use case.
- [ ] Verify Edge WebView2 remains available if required by installed applications.
- [ ] Confirm Reserved Storage state.
- [ ] Confirm Sysprep completed successfully.
- [ ] Run component-store health/size checks after final cleanup.
- [ ] Verify the final `install.wim`, `install.esd`, and ISO.
- [ ] Install the ISO in a VM and complete an end-to-end OOBE/first-login test before physical deployment.

## 📅 Annual Image Refresh

This workflow is maintained as a **yearly Windows 10 image refresh**. Update the `IMAGEYEAR` variable and output filenames when creating the next reference image.

The current release convention is:

```text
Windows 10 22H2
Reference year: 2026
```

---

### References

- [Microsoft — Features on Demand](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/features-on-demand-v2--capabilities)
- [Microsoft — Language Features on Demand](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/features-on-demand-language-fod)
- [Microsoft — Customize Windows RE](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/customize-windows-re)
- [Microsoft — Windows 10 lifecycle / ESU](https://learn.microsoft.com/en-us/windows/release-health/windows-message-center)
- [UUP dump — Windows 10 22H2](https://uupdump.net/known.php?q=category:w10-22h2)
- [Intel — Wi-Fi 6E AX210 Downloads](https://www.intel.com/content/www/us/en/products/sku/204836/intel-wifi-6e-ax210-gig/downloads.html)

---

> **Project note:** This is a personal build log and reference workflow. Keep a known-good source image and validate every yearly revision before relying on it.
