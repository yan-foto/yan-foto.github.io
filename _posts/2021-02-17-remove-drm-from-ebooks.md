---
layout:     post
title:      Removing DRM from E-Books on Linux
date:       2021-02-17 12:30:19
summary:    I used to remove DRMs from my E-Books using Calibre on MacOS because there is no Adobe Digital Editions on Linux. Now I've found way to solve this on Linux!
categories: linux drm wine
---

**tl;dr**
> We're going to install an ancient version of "Adobe Digital Editions" in wine and use some helper scripts
from the [DeDRM tools](https://github.com/apprenticeharper/DeDRM_tools) repository to remove DRM from E-Books.

## Introduction

Do you remember the time when you used to buy something and then *own* it? Well I do. That's exactly why the idea of putting DRM
on digital products that I have bought and own (in contrast to DRM on for example a rented movie) is most annoying to me. I don't pirate
any kind of art, I buy it if I can and when I buy it I want to do whatever I want to do with, so the first thing I do is to remove DRMs.
I used to do it using a [Calibre plugin](http://dedrm.com/) on MacOS.
The trick was to use an ancient version of Adobe Digital Edictions (ADE) (v 2.0.1).
As I migrated back to Linux, I wanted to have the same convenience.
But I had two problems: i) I didn't want to pollute my local machine with wine, and ii) it turned out that proper configuration of Calibre
to use ADE keys (running under wine) is not working out of the box.
My first intuition was to install wine in a virtual machine (VM) and use it only for DeDRM purposes.
I started with [Gnome Boxes](https://help.gnome.org/users/gnome-boxes/stable/) to create and manage [QEMU](https://www.qemu.org/) VMS.
It also turned out to be a pain: spice sharing didn't work properly and I didn't find a quick way to conserve Gnome VM boxes (for backup/migration purposes)
without having to copy different files from different directories etc.
So I ended up with doing everything from the scratch!

## Prepare qemu image

We start with creating a QEMU image to install Ubuntu, run ADE inside wine, and finally DeDRM our E-Books:

```bash
qemu-img create -f qcow2 ade.qcow2 10G
```

We move on to boot from an Ubuntu image and install the system:

```bash
 qemu-system-x86_64 -boot d -cdrom ~/Downloads/ubuntu-20.04-desktop-amd64.iso \
    -m 2048 -smp 8 -cpu host -enable-kvm -hda ade.qcow2
```

Note that I use a `x86_64` system instead of a 32 bit version which would theoretically suffice for Wine and ADE: **DON'T EVEN THINK ABOUT 32 BIT**. I went
down that road and it was all pain!
Here I use Ubuntu 20.04 LTD beause it just works! I tried other distributions and older version and the result was a mess (maybe I missed something!).
After Ubuntu is successfully installed, shut down the VM.

## Preparing Ubuntu

Let's run the freshly installed Ubuntu instance:

```bash
qemu-system-x86_64 -m 2048 -hda ade2.qcow2 -smp 8 -cpu host -enable-kvm \
    -fsdev local,id=shared_dev,path=$(pwd)/share,security_model=none \
    -device virtio-9p-pci,fsdev=shared_dev,mount_tag=shared_mount
```

The second and third lines are required for sharing files between host and guest machines. We'll get back to this later.
First, we need to install [wine](https://www.winehq.org/).
Don't use the version from Ubuntu repository, but use the latest version from WineHQ repo:

```bash
sudo dpkg --add-architecture i386
wget -nc https://dl.winehq.org/wine-builds/winehq.key
sudo apt-key add winehq.key
sudo apt update
sudo apt install --install-recommends winehq-stable
```

Verify that everything went well:

```
wine --version
```

I have `wine-5.0.3` and I can assure you that it everything works on this version.
Next we will create a dedicated wine prefix only for the ADE:

```
export WINEPREFIX=~/.ade
export WINEARCH=win32
winecfg
```

When wine config dialog opens, make sure that you have selected Windows XP. You can leave the rest as is.
Now we need to download [MS .NET 3.5 SP 1](https://www.microsoft.com/en-us/download/details.aspx?id=25150) and [ADE 2.0.1](https://www.adobe.com/support/digitaleditions/downloads.html).
I assume that you have both files under `~/Downloads`.
We'll go ahead and install both in our wine prefix:

```
wine start ~/Downloads/dotnetfx25.exe
wine start ~/Downloads/ADE_2.0_Installer.exe
```

Now you must be able to run ADE. Try to login (Authorize Computer) and see if it works. In my case, everytime I navigated to the password field ADE crashed. Interestingly enough, it
turned out some of fonts are missing under wine:

```
sudo apt install winetricksWebKitGTK+
winetricks -q corefonts
```

This should solve the crashes.

## Removing DRM from file

The basic idea here is to extract the private key that ADE uses to decrypt DRM and remove the DRM permanently.
We need pyhthon for that, but we need it under wine and not Ubuntu:

```
winetricks -q python27
```

An additional package, `pyCrypto` is also required for cryptographic operations which can be downloaded [here](http://www.voidspace.org.uk/python/modules.shtml#pycrypto).
**NOTE:** this is the only version of `pyCrypto` that worked for me; in general it's a very bad idea to download a package used for cryptographic purposes from some random
source on the Internet, but since we have everything sandboxed in QEMU I didn't really care, but you might do!

```
wine start ~/Downloads/pycrypto-2.6.wine32-py2.7.exe
```

Now that we have `python` and `pyCrypto` on our windows machine, we can get starting with hacking ADE.
I have [prepared two scripts](https://gist.github.com/yan-foto/9dc0ce22ef2b1f0d9131565b60acb1c5#file-0-readme-txt-L11) to make that easier for you.
Download all three `init.sh`, `dedrm.sh` and `sources.txt` and run `init.sh` which fetches necessary python scripts and extracts
the private ADE key. This step is run inside the wine machine:

<script src="https://gist.github.com/yan-foto/9dc0ce22ef2b1f0d9131565b60acb1c5.js?file=init.sh"></script>

After running this script, two new directories are added `lib` and `conf`:

    * `lib`: contains downloaded helper scripts.
    * `conf`: contains extracted ADE key.

Now open any DRM-protected book in ADE. It will save the file under `~/Documents/My Digital Editions/`.
You can now remove the DRM using the `dedrm.sh` script:

<script src="https://gist.github.com/yan-foto/9dc0ce22ef2b1f0d9131565b60acb1c5.js?file=dedrm.sh"></script>

```bash
./dedrm.sh ~/Documents/My\ Digital\ Editions/${BOOK}
```

replace `${BOOK}` with the proper file name.

## Moving files between guest and host

Remember those lines that we ignored when starting the QEMU VM? Those lines ["create a virtual filesystem (virtio-9p-device)"](https://wiki.qemu.org/Documentation/9psetup)
that we can use as a shared point of entry.
The only thing needed to do is to run the following on the guest machine:

```bash
mkdir ~/shared
sudo mount -t 9p -o trans=virtio shared_mount ~/shared/ -oversion=9p2000.L,posixacl,msize=104857600,cache=loose
```

Now every file that you put inside `~/shared` directory on the guest machine is going to be available on the host machine
under `$(pwd)/share` (see `path` argument as part of the `-fsdev` flag we used earlier to start the VM).

