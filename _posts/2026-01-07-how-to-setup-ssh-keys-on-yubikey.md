---
layout:     post
title:      How-to Setup SSH Authentication using FIDO2 Keys (Passkeys) on a Yubikey
date:       2026-01-07 22:16:19
summary:    I wanted to secure my SSH keys using a Hardware token, a series of bugs and config complications proved that it is harder than I thought.
categories: ssh fido2 passkey webauthn ctap yubikey
---

**tl;dr**
> I used to have my SSH keys, which I used for Git or Server authentication, local in encrypted files, the classic way if you wish.
> To improve security, I decided to use a Hardware Key.
> The idea was to have one SSH key per service (better privacy), while having a single PIN for the key itself.
> The [setup](#generating-ssh-keys) was easy enough, but the [configuration](#configuring-ssh) proved to be tricky.

## Introduction
The [Secure Shell (SSH) Protocol](https://www.cloudflare.com/learning/access-management/what-is-ssh/) enables secure communication over an otherwise insecure channel.
It can be used, for example, to login into remote machines, or securely transfer data (SFTP).
A secure method to authenticate to an SSH server is the use of assymetric (aka public-key) cryptography.
This can be done the old-school way (generating keys locally) or the modern way (using a hardware token).
If you are familar with [SSH and its authentication methods](#ssh-and-its-authentication-procedure) and the [FIDO2 (aka Passkey) protocol](#fido2), you can jump write to the [next section](#generating-ssh-keys).

### SSH and its Authentication Procedure
SSH communication partners can [authenticate each other](https://www.rfc-editor.org/rfc/rfc4252#section-7) using various methods: public key, password, ….
Server authentication is commonly public-key based.
That is why you see something like the following, the first time that you are connecting to a server (here `gitgoon.dev`) over SSH:

```
The authenticity of host 'gitgoon.dev (74.208.75.195)' can't be established.
ED25519 key fingerprint is SHA256:tUgUZZr15i/de2PM6w1arztIoj5iczpznrpVkK0nixI.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

The client can choose between passwords, public keys, or any other method supported by the server.
Using passwords, i.e., "something that you know", is considered weak in case of authentication: it is prune to phishing, and if someone else get's access to it, they can authenticate as you.
A better method is authenticating using "something you have", like a private key.
It is common to generate SSH key pairs using `ssh-keygen` and copying the public part to the remote machine (e.g., using `ssh-copy-id`).
It is also recommended to encrypt private keys at-rest.

To authenticate, a client uses its private key to a create a signature that can be validated by the server using the client public key.
Successful validation equals successful authentication.

### FIDO2
FIDO2 (aka Passkey) is a modern authentication method based on public-key cryptography that can also make use of harware for improved security:

> "A passkey is a FIDO authentication credential based on FIDO standards, that allows a user to sign in to apps and websites with the same process that they use to unlock their device (biometrics, PIN, or pattern). […] With passkeys, users no longer need to enter usernames and passwords or additional factors. Instead, a user approves a sign-in with the same process they use to unlock their device (for example, biometrics, PIN, pattern)."


The term FIDO2 actually refers to two distinct protocols [Webauthn](https://www.w3.org/TR/webauthn-2/):

> "an API enabling the creation and use of strong, attested, scoped, public key-based credentials by web applications, for the purpose of strongly authenticating users."

and [CTAP2](https://fidoalliance.org/specs/fido-v2.1-ps-20210615/fido-client-to-authenticator-protocol-v2.1-ps-20210615.html):

> "an application layer protocol for communication between a roaming authenticator and another client/platform, as well as bindings of this application protocol to a variety of transport protocols using different physical media."

For our use case Webauthn and CTAP are both used in the background by the `ssh-keygen` to create Yubikey secured SSH keys.

## Generating SSH Keys
I use a Yubikey 5C as hardware token, so the following instructions are in part hardware-specific.

### Preparing the Yubikey
If this a new Yubikey or you want to reset the FIDO applet (deletes all existing passkeys), make sure that you have a secure PIN set.

```bash
# Reset FIDO applet (optional)
ykman fido reset
# Set a new PIN (default pin is '123456')
ykman fido access change-pin --pin 123456 --new-pin
```

Creating the keys and securing them using Yubikey can be done using `ssh-keygen` as usual but with types `ecdsa-sk` or `ed25519-sk`:

```bash
# '-t': key type  
# '-O': option for the security key (here to force verifying presence)
# '-C': optional comment
# TIP: there is no need to add an extra password; device PIN suffices.
# TIP: save the key files under a custom name so you can know which key
#      is on which device and for which service.
ssh-keygen -t ed25519-sk -O verify-required -C "GitGoon on Yubikey 5C"
```

This will generate a so-called [non-resident or non-discoverable credential](https://www.w3.org/TR/webauthn-2/#non-discoverable-credential) key pair.
Non-resident private keys are not stored on security key itself but on the client machine.
Consider it as a blob that can be used by the security key to derive the actual private key.
When provided to the device, it reconstrcuts the private key and generate a signature (i.e., *assertion* in Webauthn terminology).
In case of Yubikey, the device encrypts the private key (using AES) and deliver it to the client.
If you prefer a resident key, i.e., on device, you can add `-O resident` to the command above.

> I opted for non-resident keys.
> If someone wants access to my private keys they have to torture me twice: once to give them the FIDO PIN and once to give the password to decrypt my hard disk, where the encrypted key resides.
> On the downside, I can only use my Yubikey on a machine where the encrypted private key is stored.

## Configuring SSH
I expected that I insert my Yubikey, `ssh` or `git pull/push` as usual and everything works fine.
This, unfortunately was not the case.

### First issue: `agent refused operation`
Every `git` interaction with remote ended with `agent refused operation`.
It [turns out](https://wiki.archlinux.org/title/SSH_keys#agent_refused_operation) that there's a known bug in the `ssh-agent` that triggers the error when keys are generated using the `-O verify-required` option.
To take `ssh-agent` out of the equation, we can just set [`IdentityAgent`](https://man.archlinux.org/man/ssh_config.5#IdentityAgent) to `none` in the config file (`~/.ssh/config`):

```ssh
Host gitgoon.dev
	IdentityAgent none
```

Replace `gitgoon.dev` with your server name (e.g., `codeberg.org`), or `*` if you want to have it applied to all hosts.

If it still does not work, first make sure that `ssh-agent` is not running:

```bash
# '-k': "Kill the current agent (given by the SSH_AGENT_PID environment variable)."
ssh-agent -k
```

then try again.

### Second issue: Explicit identity file
I have multiple SSH keys on machine.
Because I was using a custom name for my SSH key files, `ssh` could not automatically figure out the correct key to present to the server.
At first, I wasn't aware of this and tried to debug it:

```bash
# '-T': "Disable pseudo-terminal allocation."
# '-v': "Verbose mode."
ssh -T git@codeberg.org -v
```

Beside getting blocked by the server after too many attempts, at some point I found out that I need to explicitely specify the file with the private key.
This can be defined for a host using [`IdentityFile`](https://man.archlinux.org/man/ssh_config.5#IdentityFile):

```ssh
Host gitgoon.dev
	IdentityAgent none
	IdentityFile /home/USERNAME/.ssh/NAME_GIVEN_DURING_GENERATION
```

You could also use `-i` flag with `ssh` to point to the private key, yet defining it in the config file also applies it to other commands that use SSH, e.g., `git`.

### Bonus issue: SSH privacy
When an SSH client authenticates to a server, it sends all public keys that it finds on you computer (under `${HOME}/.ssh/id_*.pub`).
This poses a privacy issue, specifically if a service publishes your SSH public keys (e.g., GitHub).
Take a look at [`whoami.filippo.io`](https://github.com/FiloSottile/whoami.filippo.io) for a background, a demo, and a solution (copied and modified from the linked repo):

```ssh
Host *
    PubkeyAuthentication no
    IdentitiesOnly yes

Host example.com
    PubkeyAuthentication yes
    IdentityFile ~/.ssh/id_rsa
```

This makes sure that no public key is presented to SSH servers by default (`PubkeyAuthentication no`) and only the public key defined in the file under `IdentityFile` is to be provided.

## Conclusion
Having SSH keys secured by a hardware device improves your security.
This how-to guide showed how we can do that using a Yubikey while addressing common pitfalls.
The same key can also be used to sign Git commits, but that's for another how-to.
