# How To Mac

A collection of useful MacOS commands and tools for hardware debugging and sysadmin operations.

```bash
                    'c.          zach@iMac
                 ,xNMM.          ---------
               .OMMMMo           OS: macOS 12.6.1 21G217 x86_64
               OMMM0,            Host: Hackintosh (SMBIOS: MacPro7,1)
     .;loddo:' loolloddol;.      Kernel: 21.6.0
   cKMMMMMMMMMMNWMMMMMMMMMM0:    Uptime: 1 hour, 23 mins
 .KMMMMMMMMMMMMMMMMMMMMMMMWd.    Packages: 288 (brew)
 XMMMMMMMMMMMMMMMMMMMMMMMX.      Shell: zsh 5.8.1
;MMMMMMMMMMMMMMMMMMMMMMMM:       Resolution: 5120x1440
:MMMMMMMMMMMMMMMMMMMMMMMM:       DE: Aqua
.MMMMMMMMMMMMMMMMMMMMMMMMX.      WM: Rectangle
 kMMMMMMMMMMMMMMMMMMMMMMMMWd.    Terminal: tmux
 .XMMMMMMMMMMMMMMMMMMMMMMMMMMk   CPU: Intel i9-10900K (20) @ 3.70GHz
  .XMMMMMMMMMMMMMMMMMMMMMMMMK.   GPU: AMD Radeon RX 5700 XT
    kMMMMMMMMMMMMMMMMMMMMMMd     Memory: 17884MiB / 32768MiB
     ;KMMMMMMMWXXWMMMMMMMk.
       .cooc,.    .,coo:.
```

## Table of Contents

<!--toc:start-->

- [Background](#background)
  - [Problem](#problem)
  - [Solution](#solution)
- [Tools](#tools)
- [Commands](#commands)
  - [Kernel Logs](#kernel-logs)
  - [Graphics Debugging](#graphics-debugging)
  - [Device Properties](#device-properties)
  - [PCI Devices](#pci-devices)
  - [NVRAM](#nvram)
  - [Kextstat](#kextstat)
  - [System Properties](#system-properties)
  - [Boot Args](#boot-args)
    - [Lilu](#lilu)
  - [PKG installers (.pkg)](#pkg-installers-pkg)

<!--toc:end-->

## Background

Building and maintaining a "Hackintosh" is a choice that many make for the cost benefit.
You are given the freedom to choose your own _supported_ components, and you just might end
up with a machine that's more capable at a fraction of the cost.  
However, this can come with the trade-off of _time_. There is an initial time-to-setup, and there is ongoing
maintainence time if you choose to upgrade MacOS.  
The way that I understand the need for this ongoing maintenance during OS upgrades is that
a custom bootloader (such as [OpenCore](https://github.com/acidanthera/OpenCorePkg)) needs
to account for various changes MacOS has made with the way that it interacts with hardware.
Specifically, this is interpreted from the [Advanced Configuration and Power Interface (ACPI) Specification](https://uefi.org/htmlspecs/ACPI_Spec_6_4_html/).

### Problem

Much of your time creating a "perfect Hackintosh" will consist of making various patches for your hardware,
including patches to ACPI tables known as `SSDTs` and patching the Mac kernel through kernel extensions, referred to as `kexts`.

Most of these patches require obtaining either the `device-id` or the `acpi-path` of the device you're targeting.  
Documentation you will find online usually refers you to use the `IO Registry Explorer` app that comes with Mac.
However, this becomes a very time-consuming process, and simply does not provide the information or data
(in the format that you need) efficiently enough, if at all.

### Solution

Curating a collection of tools, commands and processes, we can greatly speed up our time to:

- Query relevant information from the devices that we need
- Debug kernel extensions via the system log

## Tools

- Download the latest release of `Xcode` and the matching version of `Additional Tools for Xcode` from https://developer.apple.com/download/all/  
  _Note: You will need an Apple Developer account for this_  
  This will provide you with various tools, and update already-existing tools that are essential to hardware debugging on MacOS.
- Download the latest release of `gfxutil` from: https://github.com/acidanthera/gfxutil/releases  
  This is a CLI tool used for listing and filtering PCI and ACPI device paths

## Commands

### Kernel Logs

_Logs since last boot_

```bash
log show --predicate 'process="kernel"' --style syslog --source --last boot --color always --info --debug
```

_Logs since last boot, case-insensitive search for "Radeon"_

```bash
log show --predicate 'process == "kernel" && eventMessage CONTAINS[c] "Radeon"' --style syslog --source --last boot --info --debug
```

### Graphics Debugging

_Complete graphics/display info_

```bash
/System/Library/Extensions/AppleGraphicsControl.kext/Contents/MacOS/AGDCDiagnose -a
```

_Find Display Stream Compression enabled/status_

```bash
/System/Library/Extensions/AppleGraphicsControl.kext/Contents/MacOS/AGDCDiagnose -a | grep -E '(DSC Enable|DSC Status)' | cut -d : -f 4,5
```

_Get `device-id` of GPU_

```bash
GFX_PCI_NAME=$(gfxutil -f display | grep -v IGPU | awk '{ print $3 }' | rev | cut -d / -f 1 | rev) ioreg -l | grep "${GFX_PCI_NAME:-GFX0@0}" -n4 | grep "device-id" | cut -d = -f 2 | xargs | sed 's/^<//;s/>$//'
```

### Device Properties

_Show currently loaded device property data_

```bash
ioreg -lw0 -p IODeviceTree -n efi -r -x | grep device-properties | sed 's/.*<//;s/>.*//' > /tmp/device-properties.hex &&
gfxutil /tmp/device-properties.hex /tmp/device-properties.plist && bat /tmp/device-properties.plist
```

### PCI Devices

_Get PCI path to only discrete/dedicated GPUs_

```bash
gfxutil -f display | grep -v IGPU | cut -d = -f 2 | xargs
```

_Verify if PCI bridge is working for GPU, will only output if GFX0@0 has `acpi-path` property_

```bash
GFX_PCI_NAME=$(gfxutil -f display | grep -v IGPU | awk '{ print $3 }' | rev | cut -d / -f 1 | rev) ioreg -lrn "${GFX_PCI_NAME:-GFX0@0}" -k acpi-path -d 1
```

_Get formatted ACPI path for device_

```bash
ioreg -lrn ${PCI_NAME:-MCHC@0} -k acpi-path -d 1 | grep acpi-path | cut -d : -f 2 | sed 's#^/##;s#\"$##;s#/#.#g#;s#@.*.#\.#g#' | sed 's#@[0-9a-f]*\|@[0-9]*##g#'
```

### NVRAM

_Get current OpenCore version_

```bash
nvram 4D1FDA02-38C7-4A6A-9CC6-4BCCA8B30102:opencore-version | awk '{ print $2 }'
```

### Kextstat

_Show loaded kernel extensions_

```bash
kextstat
```

### System Properties

_Get Device UUID_

```bash
ioreg -rd1 -c IOPlatformExpertDevice | grep IOPlatformUUID | cut -d = -f 2 | xargs
```

### Boot Args

#### Lilu

_Enable debug mode for all Lilu-based kexts_

```bash
-liludbgall
```

### PKG installers (.pkg)

There are a few common ways of installing software/applications on Mac. One of which is a `PKG` installation.
The difference between the `PKG` installation and the more common `DMG` method is that the `DMG` method
usually packages everything it needs into a single folder and will be placed into your `Applications` folder.
The `PKG` method however contains scripts and targets for files/directories on your system that will be created.
When a `PKG` does not contain its corresponding `uninstall` script, this speaks trouble. All of those files and
directories can now be considered "dangling" if left unused, floating around on your system consuming space.

Fortunately, you can use the output of the command below to determine any files/directories to remove.

_Show files/directories created by PKG installer_

```bash
pkgutil --pkgs | grep "${PKG:-microsoft.teams}" | xargs -n1 pkgutil --files
```
