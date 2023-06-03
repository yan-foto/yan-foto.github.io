---
layout:     post
title:      Migrate Nextcloud Snap to Podman (or Docker)
date:       2023-05-08 22:30:19
summary:    I did not manage to get Nextcloud Office to run on my old "Nextcloud Box", so it was time to move on.
categories: linux tty uart udev
---

**tl;dr**
> Ages ago, you could host your own Nextcloud instance using a ["Nextcloud Box"](https://nextcloud.com/box/) which was basically a Raspi and a WD 1TB HDD put inside a custom case running Nextcloud Snap.
> Running on ARM architecture, I did not manage to get Collabora Server for Nextcloud Office up and running.
> Podman is the best alternative to Snap in running Nextcloud in containers with the option of seamless updating.

## Introduction

Nextcloud Box runs Nextcloud Snap on [Ubuntu Core](https://ubuntu.com/core).
The more that you'd use Ubuntu Core, the more it gets annoying.
For one, the only software source that you're allowed to use are snaps (no VIM for you!).
And snaps are immutable monoliths containing everything that an app need to run (e.g., database, cache, etc.) bundled together and running in its own "container".
For security reasons (?), it's impossible to customize a snap, for example, [to add a PHP module to export your Nexcloud DB from MySQL to PostgreSQL](https://github.com/nextcloud-snap/nextcloud-snap/issues/2369).
All in all, it's a "my way or the highway" mentality.

My problems started as I couldn't get Nextcloud Office up and running no matter what I tried.
I bought a Mini PC and set to install Nextcloud using Podman.

## Preparation
First, I wanted to have one system user per application that owns and manages the respective pod.
This turns out to be more complicated as it may seem.
[Chris has a good walkthrough](https://blog.christophersmart.com/2021/02/20/rootless-podman-containers-under-system-accounts-managed-and-enabled-at-boot-with-systemd/) that I just summarize here:

```bash
export SERVICE="nextcloud"
# Must have sudo rights!
adduser --disabled-password --disabled-login "${SERVICE}"
# Enable running systemd services after logout
sudo loginctl enable-linger "${SERVICE}"
# Add subuid and subgid ranges (not allocated to system users by default)
NEW_SUBUID=$(($(tail -1 /etc/subuid |awk -F ":" '{print $2}')+65536))
NEW_SUBGID=$(($(tail -1 /etc/subgid |awk -F ":" '{print $2}')+65536))
 
sudo usermod \
--add-subuids ${NEW_SUBUID}-$((${NEW_SUBUID}+65535)) \
--add-subgids ${NEW_SUBGID}-$((${NEW_SUBGID}+65535)) \
  "${SERVICE}"
```

Now that we have our system user, we can easily export everything that we need out of Nextcloud snap and copy it to its home user.
First the backup:

```bash
# Run this on the onl machine!
sudo nextcloud.export
```

The script prints out where the backup is stored and we need to copy everything to the new machine.
As I had both machines on the same network, I used `nc`:

```bash
# On the old machine (snap)
tar --zstd -H posix -c "${PATH_TO_BACKUP_DIR}" | nc -q 10 -l -p 45454

# On the new machine (podman)
# I stored everything under my home directory of the dedicated 'nextcloud' user
cd "/home/${SREVICE}"
nc -w 300 "${NEXTCLOUD_ADDR}" 454545 | tar --zstd -xv

# After This we should have the following files and directories
ls "/home/${SERVICE}"
# > apps config.php data database.sql db

# We need to rename and rearrange them as follows:
# home/
# ├─ nextcloud/
# |   ├─ config
# |   |  ├─ config.php
# |   ├─ custom_apps     <-- RENAME 'apps'
# |   ├─ data
# |   ├─ db
```

## Creating a Pod and adding containers
Now we switch to `nextcloud` user:

```bash
sudo -H -u "${SERVICE}" bash -c 'cd; bash'

# Required to use systemd
export XDG_RUNTIME_DIR=/run/user/"$(id -u)"
```

We will create a [Pod](https://developers.redhat.com/blog/2019/01/15/podman-managing-containers-pods#podman_pods__what_you_need_to_know) that houses all the containers that we need:


```bash
# NOTE: exposing 9980 is only required if you want to use Collabora!
podman pod create \
--name nextcloud \
--publish 8080:80 \
--publish 9980:9980
```

Then start a redis cache:

```bash
podman run \
--detach \
--pod nextcloud \
--name nextcloud.redis \
--restart on-failure \
--label "io.containers.autoupdate=image" \
redis:latest
```

> Note the `"io.containers.autoupdate=image"` label which is useful for [auto-updating](https://www.redhat.com/sysadmin/improved-systemd-podman) the containers.

We now setup the database.
I used [Podman Secrets](https://www.redhat.com/sysadmin/new-podman-secrets-command) to avoid passing passwords in plaintext to the DB instance:

```bash
# Let's say we have our passwords stored in *.pass text files
podman secret create nextcloud.db.root db.root.pass
podman secret create nextcloud.db.user db.user.pass
rm db.*.pass
```

we can now pass secrets to the DB container:

```bash
# Let MariaDB store the data outside the container
mkdir "${HOME}/db"
# Give access to the directory to the DB user (ID 999)
podman unshare chown -R 999:999 "${HOME}/db"
# Finally, start the container
podman run \
--detach \
--pod nextcloud \
--name nextcloud.db \
--secret nextcloud.db.root \
--secret nextcloud.db.user \
--env MYSQL_DATABASE=nextcloud \
--env MYSQL_PASSWORD_FILE=/run/secrets/nextcloud.db.user \
--env MYSQL_ROOT_PASSWORD_FILE=/run/secrets/nextcloud.db.root \
--volume "${HOME}/db":/var/lib/mysql:Z \
--restart on-failure \
--label "io.containers.autoupdate=image" \
--log-driver k8s-file \
--log-opt path="${HOME}/logs/nextcloud.db.log" \
--log-opt max-size=50mb \
mariadb:10
```

> Note: it is possible to [initialize MariaDB](https://github.com/docker-library/docs/blob/master/mariadb/README.md#initializing-a-fresh-instance) with the `database.sql` dump from our export.
I did that step manually and left it out of this guide.

> Note: I persist all log files under `~/logs`

Now the whole reason that I moved away from snaps, Collabora:

```bash
# See: https://sdk.collaboraonline.com/docs/installation/CODE_Docker_image.html
podman run \
--detach \
--pod nextcloud \
--name nextcloud.collabora \
--cap-add MKNOD \
--env aliasgroup1=https://www.example.com:443 \
--env "extra_params=--o:ssl.enable=true --o:ssl.termination=false" \
--volume /etc/localtime:/etc/localtime \
--volume /etc/timezone:/etc/timezone \
--restart on-failure \
--label "io.containers.autoupdate=image" \
--log-driver k8s-file \
--log-opt path="${HOME}/logs/nextcloud.collabora.log" \
--log-opt max-size=50mb \
collabora/code:latest
```

> Note: don't forget to replace `www.example.com` with your own FQDN.

And finally start the nextcloud container:

```bash
# Give access to the `www-data` user (ID 33):
podman unshare chown -R 33:33 "${HOME}/data"
podman unshare chown -R 33:33 "${HOME}/custom_apps"
podman unshare chown -R 33:33 "${HOME}/config"
# Star the container
podman run \
--detach \
--pod nextcloud \
--name nextcloud.core \
--secret nextcloud.db.user \
--volume "${HOME}/config":/var/www/html/config:Z \
--volume "${HOME}/data":/var/www/html/data:Z \
--volume "${HOME}/custom_apps":/var/www/html/custom_apps:Z \
--restart on-failure \
--label "io.containers.autoupdate=image" \
--log-driver k8s-file \
--log-opt path="${HOME}/logs/nextcloud.core.log" \
--log-opt max-size=50mb \
nextcloud:stable
```

Now that everything is up and running, create the systemd files:

```bash
# Create Target directory
export SYSD_TARGET="${HOME}/.config/systemd/user"
mkdir -p "${SYSD_TARGET}"
cd "${SYSD_TARGET}"
# Generate files and reload systemd daemon
podman generate systemd --new --files --name nextcloud
systemctl --user daemon-reload
# Enable the service so it starts automatically on restarts
systemctl --user enable pod-nextcloud
```

## Configuring a reverse proxy and Nextcloud
Nextcloud should now work properly.
**But it doesn't**.

### Adapt configuration
We need to adapt the configuration.
Here's my adapted `config.php`:

```php
<?php
# NOTE: change all occurences of 'www.example.com' to your own domain name.

# No need to hardcode DB password.
# We'll just read it from Podman secrets.
$db_password = trim(file_get_contents('/run/secrets/nextcloud.db.user'));

$CONFIG = array (
  'htaccess.RewriteBase' => '/',
  'memcache.local' => '\\OC\\Memcache\\APCu',
  'apps_paths' => 
  array (
    0 => 
    array (
      'path' => '/var/www/html/apps',
      'url' => '/apps',
      'writable' => false,
    ),
    1 => 
    array (
      'path' => '/var/www/html/custom_apps',
      'url' => '/custom_apps',
      'writable' => true,
    ),
  ),
  # We'll use redis container for caching
  'memcache.distributed' => '\\OC\\Memcache\\Redis',
  'memcache.locking' => '\\OC\\Memcache\\Redis',
  'redis' => 
  array (
    'host' => 'localhost',
    'password' => '',
    'port' => 6379,
  ),
  'supportedDatabases' => 
  array (
    0 => 'mysql',
  ),
  # These values are not modified (from backup)
  'instanceid' => 'REDACTED',
  'passwordsalt' => 'REDACTED',
  'secret' => 'REDACTED',
  'trusted_domains' => 
  array (
    1 => 'www.example.com',
  ),
  'datadirectory' => '/var/www/html/data',
  'overwrite.cli.url' => 'https://www.example.com',
  'dbtype' => 'mysql',
  'version' => '25.0.7.1',
  'dbname' => 'nextcloud',
  'dbhost' => '127.0.0.1',
  'dbport' => '3306',
  'dbtableprefix' => 'oc_',
  'dbuser' => 'nextcloud',
  'dbpassword' => $db_password,
  # This is the subnet of podman's default
  # network, where connections from the 
  # reverse proxy (see below) are originated.
  # See:
  # $ podman network inspect podman
  'trusted_proxies' => 
  array (
    1 => '10.88.0.0/16',
  ),
  'overwritehost' => 'www.example.com',
  'overwriteprotocol' => 'https',
  'logtimezone' => 'UTC',
  'loglevel' => 0,
  'installed' => true,
  'twofactor_enforced' => true,
  'maintenance' => false,
  # MySQL 4-byte support
  # See: https://docs.nextcloud.com/server/25/admin_manual/configuration_database/mysql_4byte_support.html
  'mysql.utf8mb4' => true,
  # Change this to your own region
  'default_phone_region' => 'DE',
  'theme' => '',
  'updater.secret' => 'REDACTED',
);
```

### Update `.htaccess`
We also need to update `htaccess` rules to to remove `index.html` from URLs :

```bash
podman exec -u www-data nextcloud.core php occ maintenance:update:htaccess
```

> Note: you need to run this every time that you update the container!

### Run Nextcloud cron jobs
While logged in as the nextcloud user, edit cron jobs:

```bash
crontab -e
```

and add the following line:

```
*/5 * * * * podman exec -u www-data nextcloud.core php /var/www/html/cron.php
```

This would run the `cron.php` required by Nextcloud every 5 minutes as `www-data` inside the Nextcloud container (`nextcloud.core`).

### Setup a reverse proxy
And finally setup a reverse proxy to connect our instance with the Internet.
I used [Caddy](https://caddyserver.com) for this and here's my config at the end of the `Caddyfile` (`/etc/caddy/Caddyfile`):

```
www.example.com {
	log {
		output file /var/log/caddy/www.example.com-access.log
	}

    # To let Apple devices discover CalDAV and CardDav without typing the whole URL
	rewrite /.well-known/carddav /remote.php/dav
	rewrite /.well-known/caldav /remote.php/dav

	# See: https://caddy.community/t/caddy-reverse-proxy-nextcloud-collabora-vaultwarden-with-local-https/12052
	# And: https://caddy.community/t/caddy-reverse-proxy-nextcloud-collabora-vaultwarden-with-local-https/12052/9
	@collabora {
		path /browser/*            # Browser is the client part of LibreOffice Online
		path /hosting/discovery    # WOPI discovery URL
		path /hosting/capabilities # Show capabilities as json
		path /cool/*               # Main websocket, uploads/downloads, presentations
	}
	handle @collabora {
		reverse_proxy https://127.0.0.1:9980 {
			header_up Host "box.quaintous.com"
			transport http {
				tls_insecure_skip_verify
			}
		}
	}

	# This block is not necessary, since Nextcloud can handle this properly.
	@forbidden {
		path /.htaccess
		path /data/*
		path /config/*
		path /db_structure
		path /.xml
		path /README
		path /3rdparty/*
		path /lib/*
		path /templates/*
		path /occ
		path /console.php
	}
	handle @forbidden {
		respond 404
	}

	# Nextcloud
	reverse_proxy localhost:8080
}
```



## Why `X` and not `Y`

* `X` = Podman / `Y` = Docker<br> 10 Euros says that sooner or later Docker will also change its licensing conditions also for private use.
* `X` = Setting up from the scrach /  `Y` = using [Nextcloud AIO](https://nextcloud.com/all-in-one/)<br>"All in One" heavily relies on Docker Compose and needs to have access to Docker socket. Having access to Docker daemon from within a container simply defeats the purpose of containerization 🤷.

