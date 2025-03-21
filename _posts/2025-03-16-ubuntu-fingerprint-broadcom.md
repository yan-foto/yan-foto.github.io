---
layout:     post
title:      Setup Broadcom Fingerprint Driver (Dell ControlVault3) on Ubuntu 24.04
date:       2025-03-12 12:30:19
summary:    Dell sells Ubuntu certified laptops, yet the latest releases are not always supported. That happened as I wanted to install Broadcom Fingerprint Drivers for my Dell Latitude.
categories: linux auth fingerprint
---

**tl;dr**
> Dell seems to be very strict when it comes to hardware support.
> Even most recently Ubuntu certified laptops from Dell only support up to the previous LTS version (22.04).
> I wanted to use Ubuntu 24.04 and still use my fingerprint reader.

```bash
# Main repo is under:
# https://packages.broadcom.com/ui/repos/tree/General/dell-controlvault-drivers
wget https://packages.broadcom.com/artifactory/dell-controlvault-drivers/brcm_linux_fp_6.1.155_6.1.028.0.tgz
tar xzvf brcm_linux_fp_6.1.155_6.1.028.0.tgz
cd brcm_linux_fp
sudo ./install.sh
```

## Introduction
My brand new Latitude 5450 is [Ubuntu certified](https://ubuntu.com/certified/platforms/14583), but only up to 22.04 LTS.
I had massive power management issues that interrupted the sleep procedures and caused battery drains.
So I updated to the latest 24.04 LTS.
Everything seemed to be working except the fingerprint reader.
So where is the driver if not in the repository?
In the following I'll describe the (long) journey that it took me to find the appropriate Broadcom ControlVault3 Driver and firmware.

## Discovering hardware
You'd imagine that it would be as easy as entering your serial number on the Dell Support website and get the hardware specs for your computer.
Instead you are provided with a list of cryptic part numbers without any indications as to what their actual purposes is.
Assuming that the reader is a simple USB deive, `lsusb` revealed the first clue:

```bash
Bus 001 Device 003: ID 0a5c:5865 Broadcom Corp. 58200
```

A quick search was not really helpful.
It just revealed that there are others who are looking for the driver as well.
The second clue was that I need a package called `libfprint-2-tod1-broadcom` (for [`fprint`](https://fprint.freedesktop.org/)), which was not available on the official repositories.
At this point, I am still not sure about the exact model, but decided to go with this package.

## Installing the Ubuntu package
Launchpad is always a good starting point to look for Ubuntu software.
So I found [a page](https://launchpad.net/libfprint-2-tod1-broadcom) that pointed to a [git repo](https://git.launchpad.net/libfprint-2-tod1-broadcom) with recent activity.
Jackpot!
Cloning and installing the driver (i.e. running `install.sh`) was, however, not successful, regardless of which branch and tag I checked out.
The installation ended with the following message:

```
Device firmware state will be automatically verified on device access. If firmware upgrade is required, the device will be unavailable for a few minutes for the duration of the firmware upgrade.
```

indicating that I need to address the device to initate firmware upgrade (if necessary).
People out there suggested running `fprintd-enroll` and let it crash (due to firmware being upgraded in the background) and run it a second time afte a few minutres to make sure that the upgrade is succeeded.
Later I found in one of the Ubuntu package issues a pointer by one of the maintainers on how to debug `fprint` daemon:

```bash
env G_MESSAGE_DEBUG=all /usr/libexec/fprintd -t
```

This rund the daemon in debug mode and let's you follow what is happening in the background, e.g., which files are actually being loaded.
This however, did not help me further for the time being.
People online pointed to [another git repository](https://git.launchpad.net/~oem-solutions-engineers/libfprint-2-tod1-broadcom/+git/libfprint-2-tod1-broadcom/) which seemed to have the same codebase but some differences.
It was here that I noted a commit under `upstream` branch with the commit message `Add new upstream 6.1.26`.
Again:

* Checkout
* `./install.sh`
* `env G_MESSAGE_DEBUG=all /usr/libexec/fprintd -t`
* `fprintd-enroll`

And again it seems that nothing is working.
So I removed the installed files using `./rm.sh` and got an error message that some files marked for removal are missing in `/var/lib/fprint/.broadcomCv3pkusFW/` directory.
Since `./install.sh` does nothing but copying the driver and firmware files, and `./rm.sh` uses a hardcoded list of files to remove, I figured out that some files are not copied to the correct destination.
A [comment on Arch Linux](https://aur.archlinux.org/packages/libfprint-2-tod1-broadcom#comment-999847) later confirmed this assumption.
So I ran `./install.sh` again and copied the missing files (from the git dir) manually:

```bash
cp -a ./var/lib/fprint/fw/cv3plus /var/lib/fprint/.broadcomCv3plusFW
```

After running the daemon again, I see some new messages about the firmware being installed.
And afterwards `fprintd-enroll` acutally asked for my finger! (on the second run to be honest).

Yet, the whole procedure seemed to be tedious and I'm starting to ask myself, where are the files in the git repo comming from?
Who are the maintainers?
I'm about to put on my tin foil hat.
It seems that these are legit people who put package proprietary code for Ubuntu on behalf of hardware vendors, i.e. Broadcom (or Dell?).
One of the maintainers even went out of his way and [uploaded a `.deb` package](https://launchpad.net/~andch/+archive/ubuntu/verify-ppa/+packages) for someone who was having difficulties getting his reader up and running.
Nonetheless, I wasn't compfortable running code that I wasn't sure of its origin on my computer.

## Broadcom official driver and firmware
I was about to give up.
But before that, I did one last search with my exact USB identifiers and ended up founding some guy on Mastodon who was [facing the same problem](https://mastodon.eddmil.es/@iMeddles/114046788648821512).
And he gave the exact pointer to the [Broadcom repository](https://packages.broadcom.com/ui/repos/tree/General/dell-controlvault-drivers) in his post.
Bingo!
The archive in the repository has the same structure as the git repo on launchpad.
Before extracting and installing official files, I first ran `./rm.sh` to remove installed files from the git repo, and also executed `rm -rf /var/lib/fprint/*` to remove any residual files.
Then, I downloaded `brcm_linux_fp_6.1.155_6.1.028.0.tgz` from the Broadcom repo, extracted it, installed it, and started the `fprint` deamon and got me an up-and-running fingerprint scanner.

At this point I could finally use my fingerprint reader without any security concerns (assuming that Broadcom is not acting evil here!).
All I had to do was to run `pam-auth-update` and select fingerprint option for authentication.
Yet, I wanted to use the fingerprint **only** after I have have logged in and not for the login itself.

## Limiting fingerprint auth only to active sessions
[GrapheneOS](https://grapheneos.org/features#two-factor-fingerprint-unlock), a hardened Android version for Pixel phones, allows deactivating fingerprint for phone unlock and use it only afterwards.
I find this approach sound, since in some jurisdictions it is allowed for the police to force unlocking devices with the owner's fingerprint (see [Germany](https://www-heise-de.translate.goog/news/Oberlandesgericht-bestaetigt-zwangsweisen-Fingerabdruck-auf-Handys-10251294.html?_x_tr_sl=auto&_x_tr_tl=en&_x_tr_hl=en-US&_x_tr_pto=wapp), for example).
Ubuntu PAM, however, has a rather all-or-nothing approach.
I found [a workaround](https://askubuntu.com/a/1414343) that seemed to be working just fine for me:

```bash
# Turn off fingerprint in PAM
pam-auth-update --disable fprintd
# Backup PAM configurations
cp -a /etc/pam.d/comm-auth{,-nofinger}
cp -a /etc/pam.d/gdm-password{,.bak}
cp -a /etc/pam.d/gdm-fingerprint{,.bak}
# Enable PAM fingerprint
sudo pam-auth-update --enable fprintd
# Remove fingerprint capability from GDM
echo '@include gdm-password' > /etc/pam.d/gdm-fingerprint
sed -i 's/comm-auth/comm-auth-nofinger/' /etc/pam.d/gdm-password
```

This would simply execute the same policies for `gdm-fingerprint` and `gdm-password` and would use the `comm-auth` policies before fingerprint was enabled.
Run `diff /etc/pam.d/comm-auth{,-nofinger}` to see which lines are actually changed.
