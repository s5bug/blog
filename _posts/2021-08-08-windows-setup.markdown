---
layout: post
title:  "A No-Bullshit Guide to Setting up Windows 10"
date:   2021-08-28 09:28:59 -0700
categories: os
tags: windows
---

## Intended Audience

The instructions for this guide are geared towards users who either:
1. Want to share as much data as possible between a Windows installation and a
   Linux installation on the same machine
2. Need to keep space on a small, fast drive that Windows boots off of
3. Both: I made this guide because I need to record videos to my SSD because
   my HDD is much too slow

If you only use one drive, and do not need to split your install across two
drives or share data between another OS, you should be able to skip most
instructions that mention `F:\`. For example, you will not need to modify the
installation location of programs. However, you will still need to modify
certain environment variables.

### Extra Goals

This guide also goes through the process of setting up a web browser, a
terminal, and a JVM manager.

### Non-Goals

This guide will not cover custom themes or any sort of modifications to system
internals. All modifications done in this guide are achieved through use of
registry modifications or official Windows tools.

### Issues or Questions

The footer of this blog contains a link to my GitHub. Open an issue on the
`blog` repository there to report issues or to ask questions.

## Prerequisites and Terms

This is a list of terms, including drive names. These names may be different
for your setup. **I will not be mentioning when you should replace a drive name
with your own. Look out for drive names in commands and variable strings.**

**(o)**
: This marks a section that is mainly opinionated. It doesn't align with the
  goals of the guide, but might be helpful for people who don't want to manage
  something when it comes up down the line.

**The Windows Machine**
: A fresh Windows 10 installation

**`C:\`**
: The boot drive/partition of Windows 10. This is the smaller/faster of the two
  drives in the two-drive setup.

**`F:\`**
: The data and programs partition. When dual-booting Linux, this is your
  "shared with Linux" drive. In this case, it must be formatted to a filesystem
  that can be used on both Windows and Linux such as
  [BTRFS](https://github.com/maharmstone/btrfs). **Do your own research.** I am
  not to be trusted with filesystems.

**Special Directories**
: These are directories that can be relocated per-user, such as Documents,
  Downloads, Desktop, Pictures, and Videos.

## First Boot: Mounting `F:\` and Moving `C:\Users`

_You can skip this section if not using `F:\`._

First, install the drivers that allow you to mount your `F:\`. Once you have
verified that `F:\` has mounted and is read-writeable, create a file at the
root, called `relocate.xml`:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<unattend xmlns="urn:schemas-microsoft-com:unattend">
  <settings pass="oobeSystem">
    <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
      <FolderLocations>
        <ProfilesDirectory>_YOUR_NEW_USERS_DIR_</ProfilesDirectory>
      </FolderLocations>
    </component>
  </settings>
</unattend>
{% endhighlight %}

Make sure to change `_YOUR_NEW_USERS_DIR_` to the desired name of your
`Users` directory, for example `F:\Win\Users`.

Open an administrator command prompt. It should open to `System32`, if not you
can `cd %WinDir%\System32\`. Once you're in `System32`, you can run

```
.\Sysprep\sysprep.exe /oobe /reboot /unattend:F:\relocate.xml
```

This command will move `C:\Users` to `_YOUR_NEW_USERS_DIR_`. When you run
this command, two things will happen:
1. `sysprep` will take a while to finish, and then reboot
2. The next boot back into Windows 10 will take a _long_ time to finish

If your BIOS is configured to boot into Windows Boot Manager first, I suggest
running `sysprep` and then going to do something else for a bit. If not, wait
for `sysprep` to reboot your machine, start the boot into Windows, and then
go do something else for a bit.

## Second Boot

### Relocating Special Directories

_You can skip this section if not using `F:\`._

The special directories in your user profile can be relocated to more helpful
locations, especially if you're sharing data with a Linux installation. For
this, right click on a special directory and go to "Properties". One of the
properties tabs should say "Location". Here you can choose the new location of
the special folder, pointing it to `F:\Documents` or `F:\Downloads`.

### Removing the `MAX_PATH` Limit 

Open an administrator PowerShell, and run

{% highlight powershell %}
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
-Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
{% endhighlight %}

### Using UTF-8 Instead of Latin-1 by Default

Open Control Panel. Select "Change date, time, or number formats" under "Clock
and Region". In the new "Region" window that opens, select the "Administrative"
tab. Click the button that says "Change system locale..." and enable the box
that says "Use Unicode UTF-8 for worldwide language support".

## Third Boot

### Install the Latest PowerShell (o)

Instructions for this lie at
[https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell-core-on-windows](https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell-core-on-windows).

### Install Chocolatey

Follow the default installation instructions on
[https://chocolatey.org/install](https://chocolatey.org/install). This should
install Chocolatey to `C:\ProgramData\chocolatey`.

If you want Chocolatey to be installed in `F:\` (you most likely do!) then you
should:
1. Move `C:\ProgramData\chocolatey` to `_YOUR_NEW_CHOCOLATEY_DIR_` (I chose
   `F:\Win\ProgramData\chocolatey`)
2. "Edit the system environment variables" then point `ChocolateyInstall` to
   `_YOUR_NEW_CHOCOLATEY_DIR_` and update the corresponding entries in
   `Path`.

### Install `gsudo` (o)

`gsudo` is the Windows equivalent of Linux's `sudo`. It lets you elevate
commands without starting a new command prompt. To install it, open an
Administrator PowerShell, and run `choco install gsudo`.

### Install a Better Terminal (o)

Windows Terminal, Hyper, Cmder, etc. all provide some nicer abstractions over
the base `cmd` and `pwsh`. If you have a favorite terminal, you should install
it now.

### Install a Browser (o)

If you prefer something other than Microsoft Edge, you might want to look at
installing it through Chocolatey.

I am a Google Chrome user, so I must run a _non_-Elevated shell (to install
Chrome to `%LocalAppData%`) and run `choco install googlechrome`.

### Install WinCompose (o)

WinCompose allows you to enter special characters without the hassle of
remembering numeric Alt-codes. It's available at
[https://github.com/samhocevar/wincompose](https://github.com/samhocevar/wincompose).

### Sycnex Debloater

Follow the instructions at
[https://github.com/Sycnex/Windows10Debloater](https://github.com/Sycnex/Windows10Debloater)
to launch `Windows10DebloaterGUI`.

Run these options:

**Remove All Bloatware**
: Disables any sort of advertising. Removes unimportant applications.

**Disable Cortana**
: Does what it says on the box.

**Uninstall OneDrive**
: Completely removes OneDrive support from Explorer. Will create a backup of
  your OneDrive files on your Desktop.

### Shutup10

This is a tool that fine-tunes the debloating done by Sycnex Debloater. It's
available to download from
[https://www.oo-software.com/en/shutup10](https://www.oo-software.com/en/shutup10).

You will want to choose "Actions" and then "Apply only recommended settings".

Note: This will disable clipboard history. If you're like me and want Clipboard
History, scroll down to "Activity History and Clipboard" and disable (make the
switches red) the two options that mention "storage of clipboard history".

"File" and "Exit" Shutup10. It will prompt you to reboot.

## Extra Utilities

### Git

You most likely want to install Git on Windows without installing the entirety
of Vim and Git Bash. Git for Windows provides a distribution called "MinGit"
that does exactly this.

Navigate to the latest release of Git for Windows:
[https://github.com/git-for-windows/git/releases/latest](https://github.com/git-for-windows/git/releases/latest).
Download the latest version of `MinGit` that does **not** mention `busybox`,
and is `64-bit`. (At the time of writing, this is
`MinGit-2.32.0.2-64-bit.zip`.)

Extract the contents to a folder called `Git` in your `%LocalAppData%`, such
that `git` can be found at `%LocalAppData%\Git\cmd\git.exe`. Then add the full
path to the `cmd` folder to your User `Path` environment variable.

### Java

**As of currently, Coursier doesn't work. Use
[Jabba](https://github.com/shyiko/jabba) instead.**

[Coursier](https://get-coursier.io/) ships with a Java manager. However, the
instructions given on the Coursier website for installing the CLI do not work
at the time of writing. Instead, open a PowerShell window and download
`cs.exe`:

{% highlight powershell %}
Invoke-WebRequest -Uri "https://git.io/coursier-cli-windows-exe" -OutFile "cs.exe"
{% endhighlight %}

Try to run `.\cs.exe`. If you get a VCRUNTIME140.DLL missing error, download
the `vc_redist.x64.exe` from
[https://www.microsoft.com/en-us/download/details.aspx?id=52685](https://www.microsoft.com/en-us/download/details.aspx?id=52685).

## Known Bugs

These are most likely results of moving things to `F:\`, and wouldn't occur
normally, however
[this kind of set-up is acknowledged by Microsoft](https://docs.microsoft.com/en-us/troubleshoot/windows-server/user-profiles-and-logon/relocation-of-users-and-programdata-directories)
and should be considered valid.

Eventually I'll get issues / bug report numbers open for all of these.

### Coursier

Attempting to let Coursier install a JDK results in an Access Denied Exception.

### Discord

Joining voice channels in the Discord Stable desktop client will hang the
entire computer, rendering it unable to shutdown properly.

The Discord Canary desktop client thinks that it is running in an unsupported
web browser and will not let you unmute or share your screen.

### Jabba

Bug report: [https://github.com/shyiko/jabba/issues/804](https://github.com/shyiko/jabba/issues/804)

Jabba assumes that `Users` lives in `C:\`, so it doesn't properly load itself
on the next time PowerShell is launched. This can be fixed by modifying
`Documents\PowerShell\Microsoft.PowerShell_profile.ps1`.

### Windows Terminal

Bug report:
[https://github.com/microsoft/terminal/issues/10899](https://github.com/microsoft/terminal/issues/10899)

The Windows Terminal will not run, citing a failure to find a drive (although
which drive that is is not mentioned).
