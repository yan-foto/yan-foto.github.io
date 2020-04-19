---
layout:     post
title:      2FA Linux Smart Card Authentication (PAM + PKCS#11)
date:       2020-04-19 6:30:19
summary:    Are you paranoid enough to add smart card authentication as a second factor to your linux login (next to your password protected bios and encrypted file system)? Well you've come to the right place.
categories: linux authentication pam pkcs11
---

## Introduction
We use smart cards at our company for all kinds of authentication, document signing, and encryption.
After moving back to Linux (Debian + Gnome) from Mac, I thought adding a new layer of security would be a neat idea.
I was sure that there's an easy way to do this under Linux, yet I was more sure that the documentation would be missing!
And yes, I was right both times.

## tl;dr
We configure [PAM](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/managing_smart_cards/pluggable_authentication_modules) to enforce smart card authentication in addition to the standard password prompt as second factor authentication.
You need to have a smart card (with valid keys) and a PKCS#11 module to read your card (either [OpenSC](https://github.com/OpenSC/OpenSC/wiki) or one from card's vendor).

## Background
### Smart Cards
A smart card is a cryptographic token which uses assymetric cryptography for authentication, encryption, and signing (non-repudiation).
Since you've made it to this blog, I'll just assume that you already own one and know what it's good for!

### PKCS#11
[Public Key Cryptography Standards (PKCS)](https://web.cs.ship.edu/~cdgira/courses/CSC434/Fall2004/docs/course_docs/IntroToPKCSstandards.pdf) are a set of standards which aim to ease the development and deployment of applications based on public key cryptogaphy.
At the moment, there are 15 standards numbered PKCS#1 to PKCS#15 (some are withdrawn).
Here, we only care for [PKCS#11](https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=pkcs11):
> standard for cryptographic tokens controlling authentication information (personal identity, cryptographic keys, certificates, digital signatures, biometric data)

You can consider PKCS#11 to be the technical specification/implementation of smart cards.

### Pluggable Authentication Module (PAM)
Let's just borrow the defintion of PAM from [linux.com](https://www.linux.com/news/understanding-pam/):
> PAM is a collection of modules that essentially form a barrier between a
service on your system, and the user of the service. The modules can have
widely varying purposes, from disallowing a login to users from a particular
UNIX group (or netgroup, or subnet…), to implementing resource limits so
that your ‘research’ group can’t hog system resources.

PAM is available on a number of commercial and free Linux distributions.
Take a look under `/etc/pam.d/` to see if your system is already using PAM or not.

## Preparation
First we need to make sure that the smart card can be accessed correctly.
I just suppose that [OpenSC](https://github.com/OpenSC/OpenSC/wiki) is already installed.

```bash
pkcs11-tool --show-info
```

If you get `No slot with a token was found.`, but the card is inserted and the reader is shown, OpenSC does not recognize your card and you might need a dedicated PKCS#11 module for your card:

```bash
pkcs11-tool --module /PATH/TO/MODULE --show-info
```

If you see token information (label, flags, etc.), you're good to go.

## System Configuration
Configuring PAM is fairly easy.
What takes more time is configuring the PAM module that uses PKCS#11 for authentication.
Luckily, OpenSC provides such module and we just need to take care of the declarative configuration.

### PAM PKCS#11 Module
OpenSC provides a [PAM login modules for PKCS#11](https://github.com/OpenSC/pam_pkcs11) (`libpam-pkcs11`) which first need to be configured, before we get to PAM settings.
An example configuration is delivered with `libpam-pkcs11` (search for `pam_pkcs11.conf.example`) which needs to be copied to `/etc/pam_pkcs11` (without the `.example` suffix).
This file contains:
  
  1. General settings (e.g., debug config)
  2. (Multiple) PKCS#11 module configurations
  3. Mapper configurations

You might need to read [the manual](https://opensc.github.io/pam_pkcs11/doc/pam_pkcs11.html) for detailed explanation, here we limit ourselves to module and mapper configurations.

If your're lucky enough and your card works outt of the box with OpenSC (see previous section), you just need to take care of minor changes to module configuration and define mappers.
My card did not work out of the box and I ended adding the following module definition:

```
pkcs11_module mymodule {
  module = /PATH/TO/PROPRIETARY/MODULE;
  description = "My pkcs#11 module";
  slot_description = "none";
  support_threads = false;
  ca_dir = /etc/pam_pkcs11/cacerts/ca-chain.pem;
  cert_policy = ca,signature;
  token_type = "Smart card";
}
```

I also changed the value of `use_pkcs11_module` to `use_pkcs11_module = mymodule;`, so by default my PKCS#11 and not the one from OpenSC is used.
What matters here are `module`, `ca_dir`, and `cert_policy`:

  * `module` is set to the PKCS#11 module path
  * `ca_dir` is set to the certificate chain file
  * `cert_policy` is set to cert chain and signature verification

This configuration uses my proprietary module to access the smart card, validates the certificate chain using the file given in `ca_dir` **and** verifies the signature.
For some reason, having the CA certificates separately did not work for me so I concatenated all intermediate CA certs (including the root) into a single file (`ca-chain.pem`) and used it instead.
Yes, I know it says `ca_dir` but believe you me, `openSSL` (used under the hood for validations) also accepts a single file as argument.
Finally, note that setting `cert_policy` to `none` is a bad idea: anyonen can buy a blank smart card and try to spoof your ID and get through with it.

The last step is to setup mappers before configuring PAM.
A mapper simply defines how to map a certificate stored on a smart card to a user name on your machine.
I used both `cn` (common name) and `email` mappers:

```
use_mappers = cn, mail;

mapper cn {
      debug = false;
      module = internal;
      ignorecase = true;
      mapfile = file:///etc/pam_pkcs11/cn_map;
}

mapper mail {
      debug = false;
      module = internal;
      mapfile = file:///etc/pam_pkcs11/mail_map;
      ignorecase = true;
      ignoredomain = false;
}
```

Now you just need to create and fill two mapping files `/etc/pam_pkcs11/cn_map` and `/etc/pam_pkcs11/mail_map` using the simple `VALUE -> USER` format.
For example, you can have the following for email mapping for user `jsmith` if the certificate has `john.smith@example` in its subject alternative name (SAN):

```
john.smith@example.com -> jsmith
```

If you need to read the certificate from the smartcard before setting up mappers, you first need to find out the ID of desired certificate on your card (a card can carry multiple objects):

```bash
# use with --module flag in needed
pkcs11-tool --list-objects
```

The fetch the certificate, pipe it to `openSSL` and voila:

```bash
pkcs11-tool --read-object --type cert --id ID_FROM_PREV_STEP |
  openssl x509 -inform der -noout -text
```

You can also use this to verify the cert against the certificate chain that you created earlier:

```bash
pkcs11-tool --read-object --type cert --id ID_FROM_PREV_STEP |
  openssl x509 -inform der |
  openssl verify -CAdir /etc/pam_pkcs11/cacerts/ca-chain.pem
```

**Note**: proper flag for the last `openssl` would be `CAfile`, I just used `CAdir` to show that it also works if a file (and not a directory) is passed.

Now that you have everything configured, you can verify that you setup works properly.
Luckily, you can simulate the login procedure using `pklogin_finder`:

```bash
pklogin_finder debug
```

This would automatically read your configuration (you can also set it explicitly using `config` flag), simulate the login procedure, and let you know if everything works properly.

### Configuring PAM
I really don't want to get into [PAM confguration files](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/managing_smart_cards/pam_configuration_files), but you can see some examples under `/etc/pam.d/`.

Using PAM with the smart card with the configuration from previous section is as easy as adding a single line to your PAM configurations:

```
auth	required	pam_pkcs11.so
```

For console logins, for example, you can add this line to `/etc/pam.d/login`.
I use gnome and added this line to `/etc/pam.d/gdm-password`:

```
#%PAM-1.0
auth    requisite       pam_nologin.so
auth	required	pam_succeed_if.so user != root quiet_success
@include common-auth
auth	required	pam_pkcs11.so
auth    optional        pam_gnome_keyring.so
@include common-account
# SELinux needs to be the first session rule. This ensures that any 
# lingering context has been cleared. Without this it is possible 
# that a module could execute code in the wrong domain.
session [success=ok ignore=ignore module_unknown=ignore default=bad]        pam_selinux.so close
session required        pam_loginuid.so
# SELinux needs to intervene at login time to ensure that the process
# starts in the proper default security context. Only sessions which are
# intended to run in the user's context should be run after this.
session [success=ok ignore=ignore module_unknown=ignore default=bad]        pam_selinux.so open
session optional        pam_keyinit.so force revoke
session required        pam_limits.so
session required        pam_env.so readenv=1
session required        pam_env.so readenv=1 envfile=/etc/default/locale
@include common-session
session optional        pam_gnome_keyring.so auto_start
@include common-password
```

Everytime I login, after entering my password it checks for the smart card and asks for my pin.
Voilà! 2FA for my user over GUI login!