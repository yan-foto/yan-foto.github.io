---
layout:     post
title:      Migrating managed Wordpress instances to NGINX + PHP-FPM
date:       2020-12-25 6:30:19
summary:    As my managed wordpress services got too restrictive, it was time to move to my own VPS.
categories: linux wordpress php nginx php-fpm
---

**tl;dr**
> This post documents how I migrated my managed wordpress instances to my own VPS using [PHP-FPM](https://php-fpm.org/) and [NGINX](https://www.nginx.com/) in case I need to do it again!

## Introduction

Some years ago, I got an offer for managed wordpress that I could not refuse: unlimited instances, disk space, traffic, databases, etc.
I was fairly happy until I wanted to share a MySQL database _outside_ the service provider's private network.
Well, I couldn't, because I didn't have access rights to configure the firewall.
So I ordered the cheapest SSD VPS by [contabo](https://contabo.com/?show=vps) as an alternative.
I chose PHP-FPM for security reasons, e.g., containing script execution to a single user.

**NOTE:** WP [provides guidance](https://wordpress.org/support/article/moving-wordpress/) on how to migrate a WP instance.
Here, following assumptions are made:
  * A single WP site is being migrated
  * Domain name stays the same
  * `SSH` access to server is given

## Prepare new server

For a debian 10, that I'm using, at least the following packages need to be installed:

```bash
apt install \
  php7.3-fpm \
  php7.3-mysql \
  nginx \
  mariadb-common
```

## Backing up old instance

Although there's an official WP guide on [how to backup data](https://wordpress.org/support/article/wordpress-backups/), I wanted to have it automated and uncomplicated.
The result was a single bash script `wp-backup.sh`:

<script src="https://gist.github.com/yan-foto/48d6605b42452b55f2fd5830463a6a18.js?file=wp-backup.sh"></script>

Now you can backup as follows:

```bash
# Log on to your server
# copy wp-script.sh over
mkdir backup && cd backup
./PATH/TO/wp-script.sh PATH/TO/WP-DIR
```

## Restoring

Log on to your new server and `scp` the backup data from previous step over into the home directory of created user:

```
scp -r OLD_SERVER:/PATH/TO/backup /home/example_com
```

To quickly restore DB and WP files as well as configuring NGINX and PHP-FPM, `wp-restore.sh` can be used:

<script src="https://gist.github.com/yan-foto/48d6605b42452b55f2fd5830463a6a18.js?file=wp-restore.sh"></script>

Restoring is as easy as:

```bash
./PATH/TO/wp-restore.sh example.com PATH/TO/BACKUP
```

This script changes the system as follows:

1. A system user corresponding to given domain name is generated, e.g., `example_com`.
2. Compressed WP files are extracted to a directory under home directory of the newly created user, e.g., `/home/example_com/example.com`
3. A new SQL user is added and is filled with the previously created dump
4. A PHP-FPM configuration file is generated and activated
5. A new NGINX block is generated and activated

## Finishing touches

You're not done before you get some fresh new certificates for your wordpress instance.
Go ahead and install [`certbot`](https://certbot.eff.org/lets-encrypt/debianbuster-nginx) and run the following:

```bash
certbot --nginx\
  -d example.com -d www.example.com\
  --server 'https://api.buypass.com/acme/directory'
```

This would fetch a new certificate from [buypass](https://www.buypass.com/), a european alternative to Let's Encrypt, and automatically configure your nginx block.