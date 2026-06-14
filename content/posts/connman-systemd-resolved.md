---
title: "Adding systemd-resolved Support to Connman"
description: "Step-by-step guide to rebuild Arch Linux connman package with systemd-resolved DNS backend for proper DNS resolution."
date: 2024-09-03T00:00:00+05:30
lastmod: 2024-09-03T00:00:00+05:30
draft: false
tags: 
    - linux
    - arch
    - network
---

**Connman systemd-resolved**: Connman is a lightweight network manager for Linux that lacks built-in systemd-resolved support. To enable it, you must rebuild the connman package from source with the `--with-dns-backend=systemd-resolved` configure flag. This allows connman to delegate DNS resolution to systemd-resolved, enabling features like DNS-over-TLS and per-link DNS configuration on Arch Linux.

This is a quick guide to enable `systemd-resolved` support for `connman` package in arch because arch wiki being arch wiki, it RTFMed the reader with no futher explanation on how to actually rebuild this package.

## Prerequisites

Install a few packages before we start rebuilding.
These are the required dependencies as per connman's [PKGBUILD](https://gitlab.archlinux.org/archlinux/packaging/packages/connman/-/blob/main/PKGBUILD?ref_type=heads#L22).
- base-devel
- devtools
- bluez
- iwd
- openconnect
- openvpn
- ppp
- wpa_supplicant
> [devtools](https://man.archlinux.org/man/devtools.7.en) is required as well since it has the tools for rebuilding the package.

Command for installing:
```
sudo pacman -S bluez \
               iwd \
               openconnect \
               openvpn \
               ppp \
               wpa_supplicant \
               devtools \
               base-devel
```

## Re-Building the Package

First, We need to get the source files of connman package from arch's gitlab.

```
pkgctl repo clone --protocol=https connman
cd connman
```

Now, Open up PKGBUILD and modify it to add `--with-dns-backend=systemd-resolved` option.

```bash
...
build() {
  cd $pkgname-$pkgver
  ./configure \
    --prefix=/usr \
    ...
    --with-dns-backend=systemd-resolved
    ...
    --enable-pie \
    --enable-iwd
  make
}
...
```

Write the changes and re-build the package.
```
makepkg --syncdeps --rmdeps
```

**`--syncdeps`**: Installs required dependencies. \
**`--rmdeps`**: Removes make dependencies after building, which are not needed.

ps. If `makepkg` fails due to PGP key not being verified, you can pass `--skippgpcheck` flag to the command.
```
==> ERROR: One or more PGP signatures could not be verified!

$ makepkg --syncdeps --skippgpcheck
```

## Installing the Modified Version

Once the building process is complete, a package file (connman-pkgver.pkg.tar.zst) will be created in the working directory.

To install it, use makepkg's `-i` flag
```
makepkg --install
```
## Final setup

After installing the modified package, setup the stub resolver as `/etc/resolv.conf`
```
ln -sf ../run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

Then, Restart `connman.service`, `systemd-resolved.service` and (if you're using it) `tailscale.service`.
```
sudo systemctl restart connman.service
sudo systemctl restart systemd-resolved.service
```

That's it. :)

## Related Reading

- [C++ Notes](/posts/cpp-notes/) — Common C++ patterns and reference
- [Gemini Gemtext](/posts/gemini-gemtext/) — Lightweight protocol for simple document publishing
- [Upcycle #1: Android Projector](/posts/upcycle-1-android-projector/) — Reviving locked-down Android devices
