# FreeBSD Bastille Jail Migration (amd64 → aarch64)

This documents how I migrated a BastilleBSD jail originally built with
**amd64** binaries so it could run correctly on an **aarch64 (ARM64)**
FreeBSD host. This preserves your jail configuration and data without
rebuilding it entirely from scratch.

------------------------------------------------------------------------

## 1. Install Bastille on the New Host

Edit config:

``` sh
doas nano /usr/local/etc/bastille/bastille.conf
```

Bootstrap Bastille and the target release:

``` sh
doas bastille bootstrap
doas bastille bootstrap 14.3-RELEASE update
```

------------------------------------------------------------------------

## 2. Export the Jail from amd64 Host

``` sh
doas bastille export wirevnet /vaultdisk/wiki_transfer/
```

------------------------------------------------------------------------

## 3. Import on aarch64 Host

``` sh
doas bastille import /vaultdisk/wiki_transfer/wirevnet_2025-07-29-093557.xz
```

------------------------------------------------------------------------

## 4. Confirm Architecture Mismatch

On the host:

``` sh
file /usr/local/bastille/jails/wirevnet/root/usr/local/bin/zsh
```

Example output:

    ELF 64-bit LSB executable, x86-64, ...

This means the jail still contains **amd64 binaries**.

------------------------------------------------------------------------

## 5. Fetch Correct Base

Download FreeBSD base for `aarch64`:

``` sh
fetch https://download.freebsd.org/ftp/releases/arm64/aarch64/14.3-RELEASE/base.txz
```

------------------------------------------------------------------------

## 6. Replace the Base System

Extract over the jail's root:

``` sh
tar -xpf base.txz -C /usr/local/bastille/jails/wirevnet/root
```

This overwrites `/bin`, `/sbin`, `/lib`, `/usr/bin`, `/usr/lib`, etc.

------------------------------------------------------------------------

## 7. Remove amd64 Packages

``` sh
rm -rf /usr/local/bastille/jails/wirevnet/root/usr/local/*
rm -rf /usr/local/bastille/jails/wirevnet/root/var/db/pkg/*
```

------------------------------------------------------------------------

## 8. Re-bootstrap `pkg`

Enter the jail:

``` sh
bastille console wirevnet
```

Inside the jail:

``` sh
ASSUME_ALWAYS_YES=yes pkg bootstrap -f
```

Check architecture:

``` sh
file /usr/sbin/pkg
# → ELF 64-bit LSB executable, ARM aarch64
```

------------------------------------------------------------------------

## 9. Reinstall Packages

Update and reinstall:

``` sh
pkg update
pkg upgrade -f -y
```

If you cleared the package DB:

``` sh
pkg install zsh bash sudo ...
```

------------------------------------------------------------------------

## 10. Verify

``` sh
uname -m
# aarch64

file /usr/local/bin/zsh
# ELF 64-bit LSB executable, ARM aarch64
```

Your jail should now run natively on aarch64.

------------------------------------------------------------------------

## ⚠️ Gotchas / Caveats

-   **Mixed-arch data:** Binaries are replaced, but architecture-neutral
    data (configs, scripts, databases) should remain safe. Always back
    up first.
-   **Custom services:** If you built services from source under amd64,
    you'll need to rebuild them for aarch64.
-   **Kernel modules:** Jails can't use host kernel modules directly,
    but if you had amd64-only kernel features tied to your setup, they
    won't carry over.
-   **Performance differences:** Some amd64-compiled ports may not exist
    on aarch64. Check `pkg search` for availability before reinstalling.
-   **Third-party software:** If you manually installed software in
    `/opt` or elsewhere, you must rebuild those for aarch64.

------------------------------------------------------------------------
