---
layout: post
title:  "A No-Bullshit Guide to Setting up Windows 10"
date:   2021-08-28 09:28:59 -0700
categories: os
tags: windows
---

## Intended Audience

The instructions for this guide are geared towards users who are comfortable
with Linux and want a useful setup, or dual-boot a Linux OS and want to share
data with it. For the latter, this guide uses `F:\` to refer to the shared
drive. If you do not need a shared drive, you can mostly ignore those
instructions.

Note: Previous iterations of this blog post included instructions on how to
move `C:\Users` to a different drive. This has since been confirmed by
Microsoft Support to create an invalid system configuration, and from
experience will break a lot of things.

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

## First Boot

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

## Second Boot

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

### Install `gsudo`

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

I am a Google Chrome user, so I run `gsudo choco install googlechrome` to
install Chrome across the whole computer instead of just at user-level.

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
  your OneDrive files on your Desktop. If you get notifications about `pwsh`
  downloading a bunch of files, then you can permanently block the app to make
  the Debloater think there are no files left to download.

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

If you want Git to be installed for all users, you can extract it to
`C:\Program Files\`, such that `git` can be found at
`C:\Program Files\Git\cmd\git.exe`. Make sure to edit
`C:\Program Files\Git\etc\gitconfig` to not be self-referential.

If not, extract `Git` to your
`%LocalAppData%`, such that `git` can be found at
`%LocalAppData%\Git\cmd\git.exe`.

Then add the full path to the `cmd` folder to
your User `Path` environment variable.

### Haskell (Stack)

The 64-bit Windows installer from
[the Stack README](https://docs.haskellstack.org/en/stable/README/) will work,
as long as you install it to a directory without spaces in the path (such as
`F:\Win\Stack\bin`, whereas an invalid location would be `C:\Program Files\`).

Once this is done, you will want to fix the `STACK_ROOT` environment variable
to point to somewhere more favorable than `C:\sr`. Pointing it to the parent
of where you installed Stack will work as long as it's in its own directory
(for example, `F:\Win\Stack`).

Next, run `stack path`. This will download some stuff to
`%LocalAppData%\Programs\stack`, but don't worry, we'll get rid of it. Once
`stack path` finishes running, look for a `config.yaml` in the folder you
installed Stack to. Edit it to add the two keys at the bottom:

{% highlight yaml %}
local-programs-path: _STACK_ROOT_\programs
local-bin-path: _STACK_ROOT_\bin
{% endhighlight %}

Replacing both instances of `_STACK_ROOT_` with your Stack root, for example

{% highlight yaml %}
local-programs-path: F:\Win\Stack\programs
local-bin-path: F:\Win\Stack\bin
{% endhighlight %}

You can then delete `%LocalAppData%\Programs\stack`. If you'd like, you can run
`stack path` again to check if the changes to `config.yaml` are valid.

### Java + Scala

If you will also need Scala tools, [Coursier](https://get-coursier.io/) ships
with a Java manager. Instead of following the instructions for using `cmd`, you
can run this command for PowerShell:

{% highlight powershell %}
Invoke-WebRequest -Uri "https://git.io/coursier-cli-windows-exe" -OutFile "cs.exe"
{% endhighlight %}

Before running `.\cs.exe`, make sure to set the `COURSIER_CACHE` environment
variable to the location of your choice: Coursier has a bug right now where it
will fail to find the default cache location
([#2031](https://github.com/coursier/coursier/issues/2031),
[#2118](https://github.com/coursier/coursier/issues/2118)). When using `F:\`,
you may want something like `F:\Win\Program Files\Coursier\Cache`. If you're
not sure, refer to the
[the Coursier cache documentation](https://get-coursier.io/docs/cache#default-location).

Try to run `.\cs.exe`. If you get a VCRUNTIME140.DLL missing error, or nothing
happens, download the `vc_redist.x64.exe` from
[https://www.microsoft.com/en-us/download/details.aspx?id=52685](https://www.microsoft.com/en-us/download/details.aspx?id=52685).

### Java (no Scala)

Install [Jabba](https://github.com/shyiko/jabba).

### JavaScript (Node + Yarn)

Install [nvm-windows](https://github.com/coreybutler/nvm-windows). Once you
have a `node` installed, install Yarn with `gsudo npm install --global yarn`.

Yarn by default will dump things in `%LocalAppData%\Yarn\`. If you use `F:\`,
you most likely don't want this. Configure Yarn to use a more appropriate
directory:

```
yarn config set prefix _YARN_ROOT_
yarn config set cache-folder _YARN_ROOT_\Cache
yarn config set global-folder _YARN_ROOT_\Data\global
```

Replacing all instances of `_YARN_ROOT_` with your desired Yarn root, for
example `F:\Win\Yarn`. You do not have to create a `YARN_ROOT` environment
variable, although you can if you would like.

Next, add the output of `yarn global bin` to your `Path`. For example, if your
output looked like

```
PS> yarn global bin
F:\Win\Yarn\bin
```

You would add `F:\Win\Yarn\bin` to your `Path`. If you created a `YARN_ROOT`
environment variable, you can substitute it here.

Note that there is currently a bug in Yarn where it will think it's running on
a Linux machine when generating wrappers
([#6295](https://github.com/yarnpkg/yarn/issues/6295)). You will need to edit
any of the `.cmd` files of installed binaries. For example, with
`yarn global add vsce`, you would want to edit the `vsce.cmd` found in the
`yarn global bin` folder from

```
@IF EXIST "%~dp0\/bin/sh" (
  "%~dp0\/bin/sh"  "%~dp0\..\Data\global\node_modules\.bin\vsce" %*
) ELSE (
  @SETLOCAL
  @SET PATHEXT=%PATHEXT:;.JS;=;%
  /bin/sh  "%~dp0\..\Data\global\node_modules\.bin\vsce" %*
)
```

to

```
@IF EXIST "%~dp0\cmd.exe" (
  "%~dp0\cmd.exe"  "%~dp0\..\Data\global\node_modules\.bin\vsce.cmd" %*
) ELSE (
  @SETLOCAL
  @SET PATHEXT=%PATHEXT:;.JS;=;%
  cmd /c "%~dp0\..\Data\global\node_modules\.bin\vsce.cmd" %*
)
```
