---
layout:     post
title:      Whitelist TTY access using udev instead of joining dialout
date:       2021-09-19 22:30:19
summary:    Having my user as part of dialout somehow gives me the creeps, yet I don't want to sudo my way to UART on my microcontrollers. So, here comes the udev!
categories: linux tty uart udev
---

**tl;dr**
> Debugging microcontrollers or even full-fledged single-board computers over UART requires you to be in the `dialout` group.
> That group, however, seems like a Pandora's box to me that can open a wide attack surface.
> So, here we're going to use [`udev`](https://www.linux.com/news/udev-introduction-device-management-modern-linux-system/) to whitelist specific vendors, devices, etc. instead of blanket access using `dialout` group.

## Introduction

I started programming [Raspi Pico](https://www.raspberrypi.org/products/raspberry-pi-pico/) a while ago using [MicroPython](https://micropython.org/) and now plain C code.
As I'm a rookie, I needed some proper way to debug my code and [UART](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter) sounded just right.
Pico uses [tinyusb](https://github.com/hathach/tinyusb) to simulate a tty over USB and it lists it under `/dev/ttyACMx` with `x` being a number, e.g., `/dev/ttyACM1`.
These devices belong by default to user `root` and group `dialout`.
If you look elsewhere, you'd most probably find the recommendation to add your user to `dialout` to be able to use these devices.
Me, on the other side, saw that as an attack surface that might get exposed if for some reason someone gains access to my user [1]: there might be other critical devies accessible through `dialout`!
So I opted for `udev` to control fine-grained access control to my devices.

## Setting up `udev`

First you need to find out the some unique property about your device.
Find your device by running `ll /dev/tty*` twice to see which new device has been added.
Say, that the device is under `/dev/ttyACM1`, you can get udev info about it and its parents using:

```bash
udevadm info --attribute-walk /dev/ttyACM1
```

For Pico I get the following attributes (an excerpt only):

```
looking at parent device '/devices/pci0000:00/0000:00:14.0/usb1/1-4':
    KERNELS=="1-4"
    SUBSYSTEMS=="usb"
    DRIVERS=="usb"
    ...
    ATTRS{idProduct}=="000a"
    ATTRS{idVendor}=="2e8a"
    ATTRS{manufacturer}=="Raspberry Pi"
    ATTRS{serial}=="E66038B713108E30"
    ...
```

I can use any of `idProduct`, `idVendor`, `manufacturer`, and `serial`  to setup a [udev rule](http://www.reactivated.net/writing_udev_rules.html) as follows:

```
KERNEL=="ttyACM[0-9]", SUBSYSTEM=="tty", ATTRS{manufacturer}=="Raspberry Pi", OWNER="$(whoami)"
```

There are a number of noteworthy items in file:

- `KERNEL` is set to `ttyACM*` and not `1-4` as we see in the excerpt above. This is because we want to modify `ttyACM1` and not its parent (`usb1/1-4`)
- Similarly as above we also set the to `tty` and not `usb`
- `manufacturer` belong to one of `ttyACM1` parents so we use `ATTRS` instead of `ATTR` to match parents' attributes
- Instead of `OWNER` (don't forget to replace it with your own username), we could also use `GROUP` to override `dialout` to another group.

finally, you can save that file under `/lib/udev/rules.d/10-local.rules` and run `udevadm control --reload` for changes to take effect. Plug out and in your usb cable and see that now you are the owner of the device!


[1] I asked on "Unix & Linux" StackExchange, if people are aware of any concrete security considerations when granting someone access to `dialout` versus using `udev` for fine-grained control.
Well, no one got back to me and my question got removed:

![udev vs dialout question](/img/udev-deleted-q.png)

