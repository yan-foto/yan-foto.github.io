---
layout:     post
title:      Hiding Homelab IP behind Tiny VPS using Wireguard
date:       2025-12-23 23:03:19
summary:    My home IP should be private. So I'm hiding it behind a public IP and tunnel the traffic through Wireguard.
categories: homelab vps wireguard privacy
---

**tl;dr**
> Exposing own's home IP address is a privacy risk.
> However, renting a regular VPS is not a sustainable solution because of the recurring costs involved.
> So I rented the cheapest tiny VPS only for its public IPv4 and redirected the traffic through a Wireguard tunnel to my homelab.

## Introduction
My homelab was exposed on the Internet through the dynamic, yet dedicated, IP address assigned to me by my ISP.
Beside crawlers, monitoring/scanning services (e.g., Censys) were scanning, recording, and publishing every piece of data, that they could scrape.
I didn't want to have my private IP being stored alongside my domain name etc.

My first idea was to move my homelab to a VPS, but i) I don't trust foreign servers for my private data, and ii) I have a relatively powerful setup that would be very expensive to replicate as a VPS.
So my solution is to hide my private IP behind a public IP and pass the traffic through to my homelab.
For this, I take the cheapest VPS with unlimited traffic and free IPv4/IPv6 addresses ([IONOS VPS XS](https://www.ionos.com/servers/vps)) and use HAProxy (as pass-through proxy) and Wireguard (to tunnel to homelab).

The involved steps can be summarized as the following:

1. Create a Wireguard tunnel between the homelab and the VPS.
2. Let HAProxy pass through all TCP traffic on ports `80`/`443` using the Proxy Protocol.
3. Configure existing reverse proxy on the homelab to accept wrapped TCP packets.

## Tunnel between Homelab and VPS
Make sure that you have [Wireguard installed](https://www.wireguard.com/install/).

On both machines create a keypair for Wireguard:

```sh
# Read/Write only for user (root here)
umask 077
# Generate keypair and store private and public keys separately
wg genkey |
  tee /etc/wireguard/priv.key |
  wg pubkey > /etc/wireguard/pub.key
```

We want homelab to dial into VPS.
Create a configuration file on the VPS under `/etc/wireguard/box.conf` (`box` can anything you want) and fill it with the following:

```toml
[Interface]
Address = 10.10.0.1/31
ListenPort = 51820
PrivateKey = CONTENT_OF_PRIV.KEY_ON_VPS

[Peer]
PublicKey = CONTET_OF_PUB.KEY_ON_HOMELAB
AllowedIPs = 10.10.0.2/32
```

Our virtual private network `10.10.0.1/31` has exactly two IPs: `10.10.0.1` for the VPS and `10.10.0.2` for the homelab.

Before continuing, make sure that port `51820` UDP is open on the VPS, and then continue to adding and setting up the interface:

```bash
# box is the name fo your config file w/o the `.conf` extension
wg-quick up box
```

On machines with `systemd`, you can also use the corresponding service:

```bash
# Start interface on boot
systemctl enable --now wg-quick@box
# Check its status
systemctl status wg-quick@box
```

For more info take a look at [Ubuntu Server documentation](https://documentation.ubuntu.com/server/how-to/wireguard-vpn/common-tasks/). 

Now move on to the Homelab and create a similar Wireguard config file (same path and name):

```toml
[Interface]
Address = 10.10.0.2/31
PrivateKey = CONTET_OF_PRIV.KEY_ON_HOMELAB
ListenPort = 51820

[Peer]
PublicKey = CONTENT_OF_PUB.KEY_ON_VPS
Endpoint = IP_OF_VPS:51820
AllowedIPs = 10.10.0.1/32
PersistentKeepalive = 25
```

Similar to the VPS, start the interface and run the following command to see if the interface is up and a connection with the peer (i.e. VPS) has been established:

```bash
wg show
```

You should also be able to ping each peer:

```bash
# On VPS pinging Homelab
ping -c 3 10.10.0.2
# On Homelab pinging VPS
ping -c 3 10.10.0.1
```

## Proxy from VPS to Homelab
Now that we have a tunnel between the two machines, we can start the proxy.
The idea is to just pass every TCP packet (no, no QUIC support for now!) to the Homelab.
At the same time we want this to be transparent to the server running on the homelab, i.e., source IP addresses are preserved and are equal to response destination addresses.

The easiest way is to use the *[Proxy Protocol](https://www.haproxy.com/blog/use-the-proxy-protocol-to-preserve-a-clients-ip-address)*: 

> It is a network protocol — developed and open sourced by HAProxy Technologies — for preserving a client’s IP address when the client’s TCP connection passes through a proxy. Without such a mechanism, proxies lose this information because they act as a surrogate for the client, relaying messages to the server but replacing the client’s IP address with their own. This distorts the logs of upstream servers because the logs incorrectly indicate that all traffic originated at the proxy.

Technically, the protocol just adds an extra header to TCP packets so the reciever knows the actual origin of the packet, even though it is relayed through the proxy. See [this blog post](https://www.haproxy.com/blog/use-the-proxy-protocol-to-preserve-a-clients-ip-address#proxy-protocol-support) for supported software.

I'm using HAProxy for this with the following configuration (under `/etc/haproxy/haproxy.cfg`):

```haproxy
global
	log /dev/log local0
	log /dev/log local1 notice
	daemon
	maxconn 256

defaults
	log global
	option tcplog
	option dontlognull
	timeout connect 5s
	timeout client	50s
	timeout server	50s

frontend http_in
	bind *:80
	mode tcp
	default_backend box_http

backend box_http
	mode tcp
	server box 10.10.0.2:80 send-proxy-v2

frontend https_in
	bind *:443
	mode tcp
	default_backend box_https

backend box_https
	mode tcp
	server box 10.10.0.2:443 send-proxy-v2
```

This is pretty straightforward: we have `frontend`s, i.e. incomming packets on the VPS, and `backend`s, i.e., our homelab, and HAProxy simply forwards wrapped TCP packets received on the VPS to homelab.

You can check the config and start HAProxy as follows:

```bash
# Check if config is correct
haproxy -c -f /etc/haproxy/haproxy.cfg
# Start the service
systemctl enable --now haproxy
systemctl start haproxy
# Check the logs (in follow mode)
journalctl -u haproxy -f
```

## Reverse Proxy on the Homelab
The heavy lifting of processing the TCP packets, managing TLS connections, etc. is done by a reverse proxy running on my Homelab. I use Caddy for this, and adding support for the Proxy Protocol is as easy as adding the following on the top of the `Caddyfile` (no, I'm not using the JSON config file):

```caddy
# Global options
{
	servers {
		listener_wrappers {
			proxy_protocol {
				allow 10.10.0.1/32
				fallback_policy reject
			}
			tls
		}
	}
}
```

That's it!
