---
layout:     post
title:      Configuring OpenVPN on OpenWRT
date:       2015-12-26 18:07:19
summary:    A quick how-to on configuring OpenVPN on OpenWRT
categories: openwrt openvpn
---

## The rationale
Instead of having every device connected to an access point establish a VPN connection, I wanted to have an access point which maintains an OpenVPN connection and tunnels all the traffic through. As an OpenWRT `n00b`, I am writing this post for myself, in case I forgot how I succeeded!

The following has been tested on the infamous (and discontinued) [TP-Link WDR4900](http://www.tp-link.com/en/products/details/cat-9_TL-WDR4900.html) V1 running [OpenWRT 15.05](https://downloads.openwrt.org/chaos_calmer/15.05/) (chaos calmer) connecting to [Private Internet Access](https://www.privateinternetaccess.com/) VPN servers.

## tl;dr
 1. Get your hands on [a router](https://wiki.openwrt.org/toh/start) supporting [OpenWrt](https://openwrt.org/)
 2. Copy all files (except `README.md`) in [this gist](https://gist.github.com/yan-foto/70107775cc609390de29) under `/etc/config` of your router
 3. Add your VPN credentials under `/etc/openvpn/pia.auth` and your certificate authority (CA) and certificate revocation list (CRL) respectively under `/etc/openvpn/pia.ca.crt` and `/etc/openvpn/pia.crl.pem`
 4. reboot and be happy

## Approach
The big picture is to configure the router so it tunnels all the web traffic of its clients through VPN. For this we will define a virtual network device which is used by [OpenVPN](https://openvpn.net/) to tunnel the data and will force all the traffic going through the router to be channeled through this interface.

## Why OpenWRT?
I used to have [dd-wrt](https://www.dd-wrt.com/site/) installed on my good old [Linksys WRT54GL](https://en.wikipedia.org/wiki/Linksys_WRT54G_series) and it nearly always did everything I wanted. But then I got to know OpenWRT developers and professional users and decided to switch to it, mainly for the following reasons:

 * [OPKG package manger](https://wiki.openwrt.org/doc/techref/opkg): just like any other package manager that one might know on linux, the OpenWRT package manager makes it pretty easy to install everything needed.
 * [Unified Configuration Interface](https://wiki.openwrt.org/doc/uci): this is one big plus for OpenWRT! Instead of trying to locate configuration files of different programms and getting to know the synstax, the UCI introduces a unified syntax for all configuration files (located under `/etc/config`)

It was also important to have a familiar environment (i.e. \*nix) for rapid development.

## Getting Ready
In the following it is assumed that the router is connected via LAN and all commands are run in OpenWRT environment. You can establish an SSH connection as follows:

```sh
ssh root@192.168.1.1
# OR even better
ssh root@openwrt
```

The only package that is required is the `openvpn`. The name may vary depending on the OpenWRT version, but it can easily be found out by running the following command:

```sh
opkg search *openvpn*
```

and installed as follows:

```sh
opkg update # make sure we are updated!
opkg install openvpn
```

## Configure: The Interface
First an interface is defined which acts as the virual network device used by OpenVPM to tunnel the traffic. To do so the following lines are to be added to `/etc/config/network`:

```
config interface 'pia'
	option ifname 'tun1366'
	option proto 'none'
```

this creates the `tun1366` virtual network device which is referred to as `pia` in other configuration files (see bellow). An example of network configuration is given [here](https://gist.github.com/yan-foto/70107775cc609390de29#file-network-L47) (you might need to read the rest to understand it better).

## Configuring OpenVPN
Now we want to have the traffic of router clients to be tunneled through the interface defined in previous section through VPN. This looks like a job for OpenVPN package installed earlier.

One bad idea is to create an `.ovpn` configuration file and start the OpenVPN service manually as follows:

```sh
openvpn --config client.ovpn
```

The better (and idiomatic) approach is to create an OpenVPN configuration under `/etc/config/openvpn`. I have created a configuration file for [PIA](https://www.privateinternetaccess.com/) clients which can be downloaded [here](https://gist.github.com/yan-foto/70107775cc609390de29#file-openvpn). This script starts when router is booted.

The OpenVPN authentication credentials are to be stored under `/etc/openvpn/pia.auth` in the following format

```
USERNAME
PASSWORD
```

You also need to have the certificate authority (CA) file and certificate revocation list (CRL) respectively available under `/etc/openvpn/pia.ca.crt` and `/etc/openvpn/pia.crl.pem` (For PIA, you can download them [here](https://www.privateinternetaccess.com/openvpn/openvpn.zip))

## Preventing DNS leaks
Unfortunately OpenWRT does not update the DNS servers (see `/tmp/resolv.conf.auto`) after connecting to VPN and still uses the name servers given by DHCP server of `wan` interface. If DNS requests are not routed through the DNS servers of the VPN provider but of the ISP, the anonymity is simply compromised.

There are a number of ways in OpenWRT to use custom DNS servers. The one that worked for me without any complications was to disable `peerdns` for `lan` and `wan` interfaces and instead manually set `dns` servers for `lan` to PIA's dns servers (`209.222.18.222` and `209.222.18.218`). These changes can be seen in the example configuration given [here](https://gist.github.com/yan-foto/70107775cc609390de29#file-network-L19)

## Internet Kill Switch
The PIA Android application has a nifty feature called the "Internet Kill Switch":

> The internet kill switch activates VPN disconnect protection. If you disconnect from the VPN, your internet access will stop working. It will reactivate normal internet access when you deactivate the kill switch mode or exit the application.

We are going to have the same feature by configuring the firewall in a manner that forwards all the traffic on `lan` to `pia` (instead of `wan`) which in turn causes the rejection of all requests if OpenVPN connection is not established.

First thing is to define a zone in `/etc/config/firewall` for the `pia` interface we defined earlier:

```
config zone
        option name             pia
        option network          'pia'
        option input            REJECT
        option output           ACCEPT
        option forward          REJECT
        option masq             1
        option mtu_fix          1
```

and the last step is to change the default `forwarding` (i.e. `lan` → `wan`) so that all `lan` traffic goes through `pia`:

```
config forwarding
        option src              lan
        option dest             pia
```

The complete firewall configuration example is given [here](https://gist.github.com/yan-foto/70107775cc609390de29#file-firewall). To read more about OpenWrt firewall configuration see [here](https://wiki.openwrt.org/doc/uci/firewall).

## Conclusion
Having your traffic tunneled through VPN has, beside anonymity, a number of advantages. In Germany, for example, the Internet providers are not hold liable for any mishandling or copyright infringement of their users (see [here](https://de.wikipedia.org/wiki/St%C3%B6rerhaftung) for more info). So, for example, if you want to provide Internet access at your shop but doesn't want to take responsibility for your customers surfing, you can just tunnel your traffic through VPN since the traffic is then technically going through another telecommunication service provider instead of your ISP. This has led [some companies](http://www.airfy.com/) to register themselves as "telecommunication service providers" and let their customers have their traffic tunneled through their VPN servers against some amount of money. Using the method provided in this article you just need a router supporting OpenWRT and a VPN subscription (e.g. PIA as cheap as ~ 4$/month) to have the same effect with just a fraction of what you would pay those companies!
