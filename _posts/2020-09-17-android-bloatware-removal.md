---
layout:     post
title:      One-liner to remove bloatware from Android
date:       2020-09-17 6:30:19
summary:    Sony does not allow removal of Facebook and Netflix from their Xperia devices; it is not hard to do it anyway!
categories: linux android shell
---

## Introduction
Recently, I bought a brand new Sony Xperia mainly because of its 24-bit DAC so I could listen to my HiRes FLAC files. It was the cheapest non-chinese vendor with dual SIM support that supports HiRes audio (wired/wireless).

## Not a novel solution but more elegant!
The problem started as I noted that my phone has Netflix and Facebook pre-installed but there is no way to remove them using the standard Android GUI. A quick search led me to [this post](https://www.xda-developers.com/uninstall-carrier-oem-bloatware-without-root-access/) by xda developers which solved my problem. In short the solution is fairly straightforward:

  1. [Install ADB and enable USB debugging](https://www.xda-developers.com/install-adb-windows-macos-linux/)
  2. Connect to your device through ADB shell
  3. Search for package names that you want to remove
  4. Use `pm uninstall` to remove packages one by one

I thought to myself "well if we are using a simple shell, we could put the last two steps together and get rid of copy/pasting". So this is the result:

```sh
pm list packages | grep facebook | sed 's/package://' | xargs -n 1 pm uninstall --user 0
```

  * `pm` stands for **package manager** and is used here to `list` all installed packages and consequently `uninstall` bloatware.
  * `grep` is used to filter only desired packages (here `facebook`).
  * `sed` is to cleanup results. `pm list` prints `package:` string as prefix for each package name.
  * `xargs` takes the results from previous commands and apply them as arguments to the given command (here `pm uninstall --user 0`). Note the usage of `-n 1` that tells `xargs` to execute `pm uninstall` for each line of result not once for all results.

**One last note:** later, I found out that someone had already proposed a [similar solution](http://disq.us/p/20bem2r) but using `td` instead of `sed`.