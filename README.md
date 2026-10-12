# kernelsu-dkms (Waydroid)

An Arch Linux packaging of [KernelSU](https://github.com/supechicken/KernelSU) as a
DKMS module for container-based Android, based on the AUR `kernelsu-dkms` recipe
by Shadichy.

Unlike the upstream AUR package, this one builds and packages the **matching
Manager APK** together with the kernel module, and installs it into Waydroid for
you. The certificate the Manager is signed with is baked into `dkms.conf`, so the
module and the Manager can never drift apart.

- Upstream source: <https://github.com/supechicken/KernelSU>
- Pinned commit: `d8de2e37a85d2c4c99bc854fb0596bb02a13eb6e`
- Manager version code: `32678`
- Architecture: **x86_64 only**
- Tested on: Arch / CachyOS, Waydroid 1.6.3, kernel 7.2.9-2-cachyos

> **Heads up:** the KernelSU fork warns that loading KernelSU on some recent
> x86_64 kernels can trigger a **kernel panic**. Read
> [the fork's own notes](https://github.com/supechicken/KernelSU) before using
> this. You want a recent kernel and a working initramfs fallback.

## What installing it does

1. Builds `kernelsu-loader` and a DKMS source tree for your kernel.
2. Builds the Android x86_64 `ksud` and the Manager from the *same* checkout.
3. Signs the Manager with your key and bakes that certificate into `dkms.conf`.
4. DKMS builds and installs the module for your running kernel.
5. Patches the Waydroid seccomp profile so the container may call `reboot(2)`,
   then installs the Manager APK and verifies the kernel recognises it.

Steps 1–4 are ordinary package/DKMS work. Step 5 runs from a systemd service
triggered by a pacman hook, and it never unloads a live module — a reboot is
always safe.

## Requirements

- Arch Linux or a derivative (CachyOS tested)
- x86_64
- Waydroid, already initialised (`sudo waydroid init`) and logged into on a
  Wayland session
- Kernel headers for your **running** kernel
  (`sudo pacman -S linux-cachyos-headers`, or whatever matches your kernel). This
  is deliberately not a dependency: DKMS packages must stay usable across kernels.
- Roughly 15 GiB free and a lot of patience. The Android SDK, NDK and a Gradle
  build are involved.
- A signing key. You do not have to create it; the build does. See
  [Signing key](#signing-key).

## Install

```bash
git clone https://github.com/marufmoinuddin/kernlsu-dkms-archlinux.git
cd kernlsu-dkms-archlinux
makepkg -s
```

`-s` installs any missing build dependencies and then runs the normal build. On
the first run it will generate your signing key for you (see
[Signing key](#signing-key)). `makepkg -si` does the same and installs the
result.

Then reboot, and make sure the module loaded:

```bash
lsmod | grep kernelsu
```

If it did not load, check why:

```bash
sudo load-kernelsu
sudo journalctl -k -b | grep -i kernelsu
```

### Using an AUR helper

```bash
yay -S kernlsu-dkms-archlinux.git   # or whatever your helper calls git URLs
```

An AUR helper will resolve `modloader` for you, since it is not in the official
repositories.

## Signing key

The module only accepts a Manager signed with the certificate it was built
against, so **the key must stay the same across rebuilds**. Losing it means the
Manager gets replaced and its app-local settings reset.

**You do not need to create this yourself.** On the first `makepkg -s` the build
handles it:

- No keystore → generates a keystore and a random password together.
- Keystore **and** password file present → reuses both, silently.
- Keystore but **no** password file → prompts once for the existing password,
  checks it actually opens the keystore, then saves it. This is the one-time
  step if you already have a key from elsewhere.

After that capture, builds are fully unattended. So the usual case is just:

```bash
makepkg -s
```

Credentials live in `~/.config/kernelsu-waydroid/`:

```
manager.p12        the private key
manager-password   its password
```

Keep that directory safe. Anyone with both files can sign a Manager your kernel
will accept. The build never copies either file into the package, the package
archive, or the build log — only the signed APK and the public certificate.

The build will **never** silently replace an existing key. If it cannot unlock
one it stops and tells you, rather than generating a password that does not
match and locking you out.

### Unattended builds (CI)

If there is no terminal, the prompt is skipped. If a keystore exists without its
password file the build then fails safely. Supply the password explicitly:

```bash
KSU_SIGNING_PASSWORD='...' KSU_ALLOW_PASSWORD_PROMPT=0 makepkg -s --noconfirm
```

Or write the password file once and let the build reuse it.

### Custom locations

```bash
KSU_KEYSTORE=/path/to/manager.p12 \
KSU_PASSWORD_FILE=/path/to/manager-password \
KSU_KEY_ALIAS=my-alias \
makepkg -s
```

### Back from an old manual setup

If you previously generated the key by hand, move the password file to the name
the build expects (`password` → `manager-password`) or point `KSU_PASSWORD_FILE`
at it. Otherwise you will be asked for the password once.

## Android SDK

The build needs SDK platform `android-37.0`, build-tools `37.0.0`, NDK
`29.0.14206865` and CMake `3.22.1` in an SDK directory. Default:
`~/Android/Sdk`, override with `KSU_SDK`.

If components are missing the build stops and tells you exactly what to install.
To let it install them itself (this accepts the SDK licence prompts for you):

```bash
KSU_SDK_AUTO_PROVISION=1 makepkg -s
```

## When the integration does not run

`kernelsu-waydroid-apply` needs a logged-in Wayland user and a module built for
the running kernel. If either is missing at install time it **defers** instead of
failing, and a timer retries every 15 minutes. To run it by hand:

```bash
sudo systemctl start kernelsu-waydroid-apply.service
sudo journalctl -u kernelsu-waydroid-apply -f
```

An "Existing KernelSU keystore has no password file" style error here means the
capture step never ran; see [Signing key](#signing-key).

Backups and logs from every run are kept under:

```
/var/lib/kernelsu-waydroid/backups/
```

### Replacing the Manager with a different key

If you rebuild with a new signing key, the installed Manager will not match. The
service refuses to replace it by default, because that resets its app-local
settings. To allow it, edit `/usr/lib/systemd/system/kernelsu-waydroid-apply.service`:

```ini
[Service]
Environment=KSU_ALLOW_MANAGER_REPLACE=1
```

Then `sudo systemctl daemon-reload`.

## Uninstalling

```bash
sudo pacman -Rns kernelsu-dkms
```

Then restore the seccomp profile if you want it back:

```bash
sudo cp /var/lib/kernelsu-waydroid/backups/<run-id>/seccomp \
        /var/lib/waydroid/lxc/waydroid/waydroid.seccomp
sudo systemctl restart waydroid-container
```

You may also need to remove the Manager inside Waydroid
(`waydroid shell pm uninstall me.weishu.kernelsu`).

## Troubleshooting

**`Conflicting DKMS override /etc/dkms/kernelsu*.conf`**
An override in `/etc/dkms/` wins over the packaged `dkms.conf` and would build a
module that rejects the packaged Manager. Remove it:

```bash
sudo rm /etc/dkms/kernelsu-*.conf
sudo dkms build -m kernelsu -v 3.3.0+77+gd8de2e37 -k "$(uname -r)" --force
sudo dkms install -m kernelsu -v 3.3.0+77+gd8de2e37 -k "$(uname -r)" --force
sudo depmod -a "$(uname -r)"
```

**`Header mismatch` / build fails on kernel headers**
Your installed headers do not match the running kernel. Install the headers for
the kernel you are *running*, then reboot. This package never reboots for you.

**The Manager installed but the kernel does not see it**
```bash
sudo journalctl -k -b | grep -i "install fd for manager"
```

**`Loading KernelSU may panic`**
Do not force it. Reboot and confirm your initramfs works first:
`sudo mkinitcpio -P`.

## Notes

- Only the running kernel is explicitly built. Other kernels rebuild via the
  normal DKMS hooks, but are not validated by this package.
- No Android app is granted root automatically. You grant it in the Manager.
- The fork is GPL-2.0-only for kernel files and GPL-3.0-or-later for the rest.

## Credits

- [supechicken/KernelSU](https://github.com/supechicken/KernelSU) — the fork
- [tiann/KernelSU](https://github.com/tiann/KernelSU) — upstream
- [Shadichy](https://aur.archlinux.org/packages/kernelsu-dkms) — AUR `kernelsu-dkms`
