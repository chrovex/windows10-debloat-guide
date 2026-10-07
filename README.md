# Stage 0 - Prerequisites

## Disclaimer
This README is a note rather than a script for my process Windows 10 optimization. The codes should be treated
as reminders. Most time manual action is needed, so please DO NOT directly paste-and-run. `*` should NOT be 
treated as wildcard, it just mentioned you to check the result or files under directory. For the debloat parts,
you need to think and chooce wisely. Make sure you understand before press enter. I am not responsible for the
outcome. Do not use these on your production devices. 

## Variables
Change this:
```
set "WORKDIR=D:\win10\work"
set "VERSION=OSCurrent202610"
```

Pre-defined variables:
```
set "USERDIR=%WORKDIR%\user"
set "INSTALL_MOUNTDIR=%WORKDIR%\mount\install"
set "RE_MOUNTDIR=%WORKDIR%\mount\winre"
```

## Source Readiness
Windows 10 22H2 (19045) Feature update ISO:
https://uupdump.net/known.php?q=category:w10-22h2

Windows 10 Langpack and FoD DVD:
https://learn.microsoft.com/en-us/azure/virtual-desktop/language-packs

Intel WiFi AX210 Drivers:
https://www.intel.com/content/www/us/en/products/sku/204836/intel-wifi-6e-ax210-gig/downloads.html

Oscdimg:
```
winget install -e --id Microsoft.OSCDIMG
```

## Preparation
Recommend to turn off antivirus software to accelerate the process, do at your own risk.
You need run cmd as administrators.

Copy `install.wim` and `boot.wim` in `dvd\sources` to USERDIR.

# Stage 1 - Offline Tweaks

## Mount Wim
List all indexes:
```
dism /Get-WimInfo /WimFile:%USERDIR%\install.wim
```

Mount wimfile:
```
dism /Mount-Image /ImageFile:%USERDIR%\install.wim /Index:1 /MountDir:%INSTALL_MOUNTDIR%
dism /Get-MountedWimInfo
```

## WinRE Rebuild
Copy `%INSTALL_MOUNTDIR%\Windows\System32\Recovery\winre.wim` to USERDIR.

Mount WinRE wimfile:
```
dism /Get-WimInfo /WimFile:%USERDIR%\winre.wim
dism /Mount-Image /ImageFile:%USERDIR%\winre.wim /Index:1 /MountDir:%RE_MOUNTDIR%
dism /Get-MountedWimInfo
```

Add fonts:
```
dism /Image:%RE_MOUNTDIR% /Add-Package /PackagePath:%WORKDIR%\integrate\winre\WinPE-FontSupport-ZH-CN.cab
dism /Image:%RE_MOUNTDIR% /Get-Packages | findstr /i "FontSupport"
```

Cleanup, commit changes and export:
```
dism /Image:%RE_MOUNTDIR% /Cleanup-Image /StartComponentCleanup /Resetbase
dism /Unmount-Image /MountDir:%RE_MOUNTDIR% /Commit
dism /Export-Image /SourceImageFile:%USERDIR%\winre.wim /SourceIndex:1 /DestinationImageFile:%USERDIR%\winre_rebuild.wim /Compress:max /CheckIntegrity
```

Copy `%USERDIR%\winre_rebuild.wim` to `%INSTALL_MOUNTDIR%\Windows\System32\Recovery\winre.wim`.

## Integrate CJK Fonts and Pinyin IME
Add Chinese support (not display language; no OCR, handwriting or TTS) and fonts:
```
dism /Image:%INSTALL_MOUNTDIR% /Add-Capability /CapabilityName:Language.Basic~~~zh-CN~0.0.1.0 /Source:%WORKDIR%\integrate\lang /LimitAccess
dism /Image:%INSTALL_MOUNTDIR% /Add-Capability /CapabilityName:Language.Fonts.Hans~~~und-HANS~0.0.1.0 /Source:%WORKDIR%\integrate\lang /LimitAccess
```

Check language support capabilities:
```
dism /Image:"%INSTALL_MOUNTDIR%" /Get-Capabilities | findstr /i "Language"
dism /Image:"%INSTALL_MOUNTDIR%" /Get-CapabilityInfo /CapabilityName:Language.Basic~~~zh-CN~0.0.1.0
```

## Integrate Drivers
Add Intel AX200/AX210 drivers:
```
dism /Image:%INSTALL_MOUNTDIR% /Add-Driver /Driver:%WORKDIR%\integrate\driver\Netwtw08.INF
```

Check driver information:
```
dism /Image:%INSTALL_MOUNTDIR% /Get-Drivers
```

## Disable Features
Generate list with all features:
```
dism /Image:%INSTALL_MOUNTDIR% /Get-Features | findstr /B /C:"Feature Name :" > %USERDIR%\feature_remove.txt
```

(Important!) Manually delete the `Feature Name : ` prefix, curate disable items before continue. Status can be check using:
```
dism /Image:%INSTALL_MOUNTDIR% /Get-Features /Format:Table
```

Disable features:
```
for /f "usebackq delims=" %P in ("%USERDIR%\feature_remove.txt") do (dism /Image:%INSTALL_MOUNTDIR% /Disable-Feature /FeatureName:%P /Remove)
```

Check features information:
```
dism /Image:%INSTALL_MOUNTDIR% /Get-Features /Format:Table
```

## Remove Capabilities
Generate list with all capabilities:
```
dism /Image:%INSTALL_MOUNTDIR% /Get-Capabilities | findstr /B /C:"Capability Identity :" > %USERDIR%\cap_remove.txt
```

(Important!) Manually delete the `Capability Identity :` prefix, curate removal items before continue. Status can be check using:
```
dism /Image:%INSTALL_MOUNTDIR% /Get-Capabilities /Format:Table
```

Remove capabilities:
```
for /f "usebackq delims=" %P in ("%USERDIR%\cap_remove.txt") do (dism /Image:%INSTALL_MOUNTDIR% /Remove-Capability /CapabilityName:%P)
```

Check capabilities information:
```
dism /Image:%INSTALL_MOUNTDIR% /Get-Capabilities /Format:Table
```

## Remove Provisioned AppX
Generate list with all appx:
```
dism /Image:%INSTALL_MOUNTDIR% /Get-ProvisionedAppxPackages | findstr /B /C:"PackageName :" > %USERDIR%\appx_remove.txt
```

(Important!) Manually delete the `PackageName :` prefix, curate removal items before continue. Status can be check using:
```
dism /Image:%INSTALL_MOUNTDIR% /Get-ProvisionedAppxPackages
```

Remove appx:
```
for /f "usebackq delims=" %P in ("%USERDIR%\appx_remove.txt") do (dism /Image:%INSTALL_MOUNTDIR% /Remove-ProvisionedAppxPackage /PackageName:%P)
```

Check appx information:
```
dism /Image:%INSTALL_MOUNTDIR% /Get-ProvisionedAppxPackages
```

Optimize appx:
```
dism /Image:%INSTALL_MOUNTDIR% /Optimize-ProvisionedAppxPackages
```

## Commit Changes and Remount
```
dism /Unmount-Image /MountDir:%INSTALL_MOUNTDIR% /Commit
dism /Mount-Image /ImageFile:%USERDIR%\install.wim /Index:1 /MountDir:%INSTALL_MOUNTDIR%
dism /Get-MountedWimInfo
```

## Remove Packages
List all visible packages:
```
dism /Image:%INSTALL_MOUNTDIR% /Get-Packages | findstr /B /C:"Package Identity :" > %USERDIR%\package_remove.txt
```

(Important!) Manually delete the `Package Identity :` prefix, curate removal items before continue. Status can be check using:
```
dism /Image:%INSTALL_MOUNTDIR% /Get-Packages /Format:Table
```

Remove package:
```
for /f "usebackq delims=" %P in ("%USERDIR%\package_remove.txt") do (dism /Image:%INSTALL_MOUNTDIR% /Remove-Package /PackageName:%P)
```

Check packages information:
```
dism /Image:%INSTALL_MOUNTDIR% /Get-Packages /Format:Table
```

## Remove Other Payloads
Edge and related files:
```
%INSTALL_MOUNTDIR%\Program Files (x86)\Microsoft\Edge\
%INSTALL_MOUNTDIR%\Program Files (x86)\Microsoft\EdgeUpdate\
%INSTALL_MOUNTDIR%\Program Files (x86)\Microsoft\Temp\
%INSTALL_MOUNTDIR%\Program Files\WindowsApps\Microsoft.MicrosoftEdge*
%INSTALL_MOUNTDIR%\ProgramData\Microsoft\EdgeUpdate
%INSTALL_MOUNTDIR%\Windows\System32\MicrosoftEdge*
%INSTALL_MOUNTDIR%\Windows\SystemApps\Microsoft.MicrosoftEdge*
%INSTALL_MOUNTDIR%\WindowsApps\Microsoft.MicrosoftEdge*
```

OneDrive and related files:
```
%INSTALL_MOUNTDIR%\Windows\System32\OneDriveSetup.exe
%INSTALL_MOUNTDIR%\Windows\SysWOW64\OneDriveSetup.exe
```

Wallpapers，lockscreen and themes:
```
%INSTALL_MOUNTDIR%\Windows\Resources\Themes
%INSTALL_MOUNTDIR%\Windows\Web\Screen
%INSTALL_MOUNTDIR%\Windows\Web\Wallpaper
```

Sample media:
```
%INSTALL_MOUNTDIR%\Users\Public\Documents
%INSTALL_MOUNTDIR%\Users\Public\Music
%INSTALL_MOUNTDIR%\Users\Public\Pictures
%INSTALL_MOUNTDIR%\Users\Public\Videos
%INSTALL_MOUNTDIR%\Windows\RetailDemo
```

Misc:
```
%INSTALL_MOUNTDIR%\inetpub
```

## Inject Audit Unattend
Create `Unattend.xml` in `%INSTALL_MOUNTDIR%\Windows\Panther\Unattend`:
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

Copy this readme.md itself to `%INSTALL_MOUNTDIR%\Windows\Panther` for VM paste convenience.


## Commit Changes and Export Wimfile
Commit changes and unmount:
```
dism /Unmount-Image /MountDir:%INSTALL_MOUNTDIR% /Commit
```

Export stage wimfile:
```
dism /Export-Image /SourceImageFile:%USERDIR%\install.wim /SourceIndex:1 /DestinationImageFile:%USERDIR%\install_stage.wim /Compress:max /CheckIntegrity
dism /Get-WimInfo /WimFile:%USERDIR%\install_stage.wim
```


# Stage 2 - Customization in Audit Mode

## Setup VM and BOOT in Audit Mode
Create VM with 35Gb vhdx, using diskpart to create EFI and SYSTEM partition, assign SYSTEM partition to V:.

Apply stage wimfile to vhdx:
```
dism /Apply-Image /ImageFile:%USERDIR%\install_stage.wim /Index:1 /ApplyDir:V:\
dism /Image:V: /Optimize-Image /Boot
```

Unattach vhdx.

Boot VM with original installation media to WinRE, assign EFI partition to S:.

Make vhdx bootable, suppose V: for SYSTEM partition:
```
V:\Windows\System32\bcdboot V:\Windows /s S: /f UEFI
```

Reboot VM, and it will enter audit mode to complete CBS pending process.

## Online Tweaks
Disable reserved storage:
```
dism /Online /Get-ReservedStorageState
dism /Online /Set-ReservedStorageState /State:Disabled
dism /Online /Get-ReservedStorageState
```

Remove Edge and OneDrive, but not Edge WebView:

- Indentify services from `sc query type=all | findstr /i "Edge"` and `sc query type=all | findstr /i "OneDrive"` manually, use `sc stop <servicename>` and `sc delete <servicename>` to clean
```
sc delete edgeupdate
sc delete edgeupdatem
```

- Delete scheduled tasks from `schtasks /query /fo LIST | findstr /i "Edge"` and `schtasks /query /fo LIST | findstr /i "OneDrive"`

- Clean Edge and Ondrive registry:
```
HKCR\Applications\msedge.exe
HKCR\CLSID\{018D5C66-4533-4307-9B53-224DE2ED1FE6}
HKCR\microsoft-edge
HKCR\microsoft-edge-holographic
HKCR\msedge*

HKLM\SOFTWARE\Microsoft\Edge
HKLM\SOFTWARE\Microsoft\EdgeUpdate
HKLM\SOFTWARE\Microsoft\OneDrive
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\Microsoft Edge
HKLM\SOFTWARE\Policies\Microsoft\Edge
HKLM\SOFTWARE\Policies\Microsoft\EdgeUpdate
HKLM\SOFTWARE\Policies\Microsoft\Windows\OneDrive
HKLM\SOFTWARE\WOW6432Node\Microsoft\Edge
HKLM\SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate
HKLM\SOFTWARE\WOW6432Node\Microsoft\OneDrive
HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\Microsoft Edge
```

Registry optimization: 
- Mimimum telemetry and diagnostic data:
```
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v AllowTelemetry /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v MaxTelemetryAllowed /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v AllowDeviceNameInTelemetry /t REG_DWORD /d 0 /f
```

- Disable Advertising ID:
```
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\AdvertisingInfo" /v DisabledByGroupPolicy /t REG_DWORD /d 1 /f
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\AdvertisingInfo" /v Enabled /t REG_DWORD /d 0 /f
```

- Block consumer features and sponsored apps:
```
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\CloudContent" /v DisableWindowsConsumerFeatures /t REG_DWORD /d 1 /f
```

- Disable Cortana:
```
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search" /v AllowCortana /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search" /v DisableWebSearch /t REG_DWORD /d 1 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search" /v ConnectedSearchUseWeb /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Search" /v AllowSearchToUseLocation /t REG_DWORD /d 0 /f
```

- Disable news and interests:
```
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\Windows Feeds" /v EnableFeeds /t REG_DWORD /d 0 /f
```

- Disable feedback notifications:
```
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection" /v NumberOfSIUFInPeriod /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection" /v PeriodInNanoSeconds /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection" /v DoNotShowFeedbackNotifications /t REG_DWORD /d 1 /f
```

- Disable activity history:
```
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\System" /v EnableActivityFeed /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\System" /v PublishUserActivities /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\System" /v UploadUserActivities /t REG_DWORD /d 0 /f
```

- Disable location services:
```
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\LocationAndSensors" /v DisableLocation /t REG_DWORD /d 1 /f
```

Service optimization to disable following (refer to BlackViper):
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

Scheduled task to disable following:
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

## CompactOS
```
compact /CompactOs:always
```

## Sysprep
Do this after the image has settled into final state:
```
%WINDIR%\System32\Sysprep\Sysprep.exe /generalize /oobe /shutdown
```

# Stage 3 - Cleanup and Make Installation Media

## Attach Vhdx
Assign SYSTEM partition to V:.

## Components Clean
Check WinSxS integrity and size:
```
dism /Image:V: /Cleanup-Image /ScanHealth
dism /Image:V: /Cleanup-Image /AnalyzeComponentStore
```

Clean components:
```
dism /Image:V: /Cleanup-Image /StartComponentCleanup /Resetbase
```

Clean and hide service pack:
```
DISM /Image:V: /Cleanup-Image /SPSuperseded /HideSP
```

Check size again:
```
dism /Image:V: /Cleanup-Image /AnalyzeComponentStore
```

## NTFS Compression Directories
```
V:\Windows\WinSxS
V:\Windows\servicing\LCU
V:\Windows\System32\DriverStore\FileRepository
V:\Windows\Installer
V:\Windows\SoftwareDistribution
```

## Remove More Files
(Important!) Remove audit mode directories:
```
V:\Windows\Panther
V:\Windows\System32\Sysprep\Panther
V:\Users\Administrator
```

Remove WinSxS backup `V:\Windows\WinSxS\Backup`.


Remove redundant files generate during customization:
```
V:\$Recycle.Bin
V:\hiberfil.sys
V:\pagefile.sys
V:\ProgramData\Microsoft\Diagnosis
V:\ProgramData\Microsoft\Search\Data
V:\ProgramData\Microsoft\Windows\WER
V:\swapfile.sys
V:\Users\*\AppData\Local\Microsoft\Windows\Explorer\*cache*.db
V:\Users\*\AppData\Local\Temp\*
V:\Windows\CSC
V:\Windows\DeliveryOptimization
V:\Windows\Downloaded Program Files
V:\Windows\LiveKernelReports
V:\Windows\Logs
V:\Windows\MEMORY.DMP
V:\Windows\Minidump
V:\Windows\Prefetch
V:\Windows\ServiceProfiles\LocalService\AppData\Local\FontCache
V:\Windows\SoftwareDistribution\DataStore
V:\Windows\SoftwareDistribution\Download
V:\Windows\System32\winevt\Logs\*
V:\Windows\Temp\*
``` 

## Capture Vhdx to Wimfile
Optimize image:
```
dism /Image:V: /Optimize-Image /Boot
```

Capture image, right after optimize:
```
dism /Capture-Image /ImageFile:%WORKDIR%\output\install_%VERSION%.wim /CaptureDir:V: /Name:"Windows 10 Pro" /Description:"Windows 10 Pro %VERSION%" /Compress:max /CheckIntegrity
dism /Get-WimInfo /WimFile:%WORKDIR%\output\install_%VERSION%.wim
```

## Make Installation DVD
Export ESD:
```
dism /Export-Image /SourceImageFile:%WORKDIR%\output\install_%VERSION%.wim /SourceIndex:1 /DestinationImageFile:%WORKDIR%\output\install_%VERSION%.esd /Compress:recovery /CheckIntegrity
```

Remove all files in dvd. Extract `dvd_structure.zip` to dvd.

Rename and copy `install.esd` or `install.wim` to `dvd\sources`. Copy `boot.wim` to `dvd\sources`.

Make installation DVD:
```
oscdimg -m -o -u2 -udfver102 -b%WORKDIR%\dvd\boot\etfsboot.com -e -bootdata:2#p0,e,b%WORKDIR%\dvd\boot\etfsboot.com#pEF,e,b%WORKDIR%\dvd\efi\microsoft\boot\efisys.bin %WORKDIR%\dvd\ %WORKDIR%\output\en_win10_%VERSION%.iso
```

Unattach Vhdx and cleanup USERDIR.

# Post Stage - Post Installation

## Deploy WinRE
```
xcopy /h C:\Windows\System32\Recovery\Winre.wim C:\Recovery\WindowsRE
C:\Windows\System32\Reagentc /setreimage /path C:\Recovery\WindowsRE /target C:\Windows
reagentc /info
```

## Disable Windows Update after all settled
Use Group Policy to disable:
```
Computer Configuration\Administrative Templates\Windows Components\Windows Update\Configure Automatic Updates
Computer Configuration\Administrative Templates\Windows Components\Windows Update\Manage end user experience
```

Optional disable service to save resources:
```
Windows Update
Background Intelligent Transfer Service
Update Orchestrator Service for Windows Update
Windows Update Medic Service (may reject as is protected)
```
