<div align="center">

<a href="https://www.omnix-linux.com">
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/omnix-banner-dark.svg">
  <source media="(prefers-color-scheme: light)" srcset="assets/omnix-banner-light.svg">
  <img alt="Omnix — FHS-compliant NixOS" src="assets/omnix-banner-light.svg" width="720">
</picture>
</a>

### NixOS reproducibility, with an ordinary `/usr/bin`.

<a href="https://www.omnix-linux.com"><img alt="Visit www.omnix-linux.com" src="https://img.shields.io/badge/Visit-www.omnix--linux.com-8B5CF6?style=for-the-badge&labelColor=5277C3" height="46"></a>

[![NixOS](https://img.shields.io/badge/NixOS-unstable-5277C3?style=for-the-badge&logo=nixos&logoColor=white)](https://nixos.org)
[![FHS 3.0](https://img.shields.io/badge/FHS-3.0_compliant-7EBAE4?style=for-the-badge&logo=linuxfoundation&logoColor=white)](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html)
[![Omarchy](https://img.shields.io/badge/Runs-Omarchy-8B5CF6?style=for-the-badge&logo=archlinux&logoColor=white)](https://omarchy.org)

[**Summary**](#executive-summary) · [**Background**](#background) · [**Why it happens**](#why-this-happens) · [**How Omnix fixes it**](#how-omnix-fixes-it) · [**Install**](#install-walkthrough) · [**Repo layout**](#repo-layout) · [**Testing**](#how-we-test) · [**Get started**](#get-started)

</div>

---

## Executive summary

**NixOS can't run most software built for "normal" Linux.** A program downloaded from a vendor, a pip wheel, an npm
package with a native binary, or an AppImage all expect to find their shared libraries in `/lib` and `/usr/lib`.
On NixOS those directories don't exist, so the program fails before its first line of code runs.

This is not a bug. It's a deliberate trade-off at the centre of NixOS's design, and it buys the reproducibility and
safe rollbacks that people choose NixOS for. But the cost is real: every NixOS user eventually hits it, the tools
that work around it have to be found and wired up by hand, and whole distributions like
**[Omarchy](https://omarchy.org)** can't run on NixOS at all.

**Omnix makes NixOS FHS-compliant.** It's a small base distribution that adds the standard `/lib64` loader, `/usr/lib`,
`/bin`, and library cache to NixOS, so prebuilt Linux software just runs. It never moves or patches the Nix store, so
every package still comes from the official NixOS binary cache, and every change can still be rolled back. On top of
the base, you pick a **flavor**, a complete desktop such as an Omarchy port or a KDE Plasma setup.

**Why now.** There's been a lot of debate about whether the NixOS project would ever work with Omarchy, given the
politics around its creator, DHH. For Omnix, it turns out not to matter. NixOS's real advantage is the public binary
cache at [cache.nixos.org](https://cache.nixos.org). Omnix builds on that cache without changing a single package,
so it doesn't need anyone's permission or any infrastructure of its own: the installer ISO is hosted on GitHub
Releases, and everything else comes straight from the NixOS cache.

---

## Background

Almost every program on Linux is **dynamically linked**: the executable file holds only the program's own code, and
it borrows common code (the C library, the C++ runtime, compression, graphics, TLS) from **shared libraries** that are
installed once and used by everything.

When you start such a program, two lookups happen before `main()` runs:

1. **Find the loader.** The executable names a *dynamic linker* to start it. On x86-64 this path is fixed in the file
   at build time: `/lib64/ld-linux-x86-64.so.2`.
2. **Find the libraries.** The loader reads the list of libraries the program needs (`libc.so.6`, `libstdc++.so.6`,
   `libz.so.1`, …) and searches for each one in a standard order: paths baked into the binary, `LD_LIBRARY_PATH`,
   the cache in `/etc/ld.so.cache`, then the default directories `/lib` and `/usr/lib`.

The [Filesystem Hierarchy Standard (FHS)](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html) is the
agreement that makes this work everywhere. Debian, Fedora, Arch, and Ubuntu all put the loader and libraries in the
same places, so one binary built on any of them runs on all of them. Software vendors rely on this. So does nearly
every shell script, which starts with `#!/bin/bash` and expects tools in `/usr/bin`.

---

## Why this happens

NixOS deliberately does **not** follow the FHS.

Instead of installing libraries into shared directories, Nix puts every package in its own directory in the
**Nix store**, named by a hash of everything used to build it:

```text
/nix/store/q4kd9…-glibc-2.40/lib/libc.so.6
/nix/store/8vbm2…-gcc-14.2.0-lib/lib/libstdc++.so.6
/nix/store/zr4c1…-zlib-1.3.1/lib/libz.so.1
```

This is the core of what makes NixOS good:

- **Many versions coexist.** Two programs can use two different versions of the same library without conflict.
- **Nothing is overwritten.** An upgrade adds new store paths instead of replacing files, so a rollback is just
  switching back to the old set.
- **Builds are pure.** A package can only see the dependencies it declared, so builds are reproducible.

Software that Nix builds itself works because Nix rewrites each binary at build time to point at exact store paths
for its loader and libraries. Software built anywhere else hasn't been rewritten. It still asks for
`/lib64/ld-linux-x86-64.so.2` and searches `/usr/lib`, and on NixOS those paths are empty or missing. There is no
single "the" `libstdc++.so.6` for the loader to find, only a dozen hash-named copies in the store, and nothing tells
it which one to use.

---

## How Omnix fixes it

The rule is **add, never relocate.** Omnix leaves `/nix/store` untouched and adds the standard paths on top of it,
using tools nixpkgs already ships:

- **`/lib64/ld-linux-x86-64.so.2`** is provided by [nix-ld](https://github.com/nix-community/nix-ld), a small shim
  that starts the real glibc loader with Omnix's library path. Unmodified binaries start, including from systemd
  services, cron, and `ssh host cmd`.
- **`/usr/lib` and `/lib`** point at a declared set of common libraries: the C/C++ runtime, compression, crypto,
  SQLite, ICU, and more. An optional **desktop preset** adds GTK, Qt, WebKitGTK, Mesa, Vulkan, X11, and audio
  libraries for Electron apps, Playwright browsers, and AppImages.
- **`/etc/ld.so.cache` and `/sbin/ldconfig`** are regenerated on every switch, so code that searches for libraries
  itself, like Python's `ctypes.util.find_library`, finds them.
- **`/bin` and `/usr/bin`** are provided by [envfs](https://github.com/Mic92/envfs), so `#!/bin/bash`,
  `#!/usr/bin/python3`, and hard-coded tool paths resolve to whatever is on `PATH`.

All of it is generated from your configuration and points through `/run/current-system`, so rolling back a NixOS
generation rolls back the library layout with it. Nix's own builds still can't see `/usr/lib`, so they stay pure.

---

## Install walkthrough

Installing Omnix takes five choices and one download. The installer is a text menu: arrow keys to move, Enter to
pick. Nothing is written to disk until you confirm at the end of step 4.

### 1. Boot the installer ISO

Download the ISO from [Releases](https://github.com/Omnix-Linux/Omnix/releases), write it to a USB stick, and boot
from it. It's the stock NixOS minimal installer plus the Omnix base, so it runs on anything NixOS supports.

Then run `sudo omnix-install`. The first thing it does is get online, because everything after this step is
downloaded. A wired connection is picked up automatically. On Wi-Fi, it opens `nmtui` so you can choose a network.

### 2. Pick a disk and a filesystem

```text
  Install to which disk? (it will be ERASED)
  ▸ /dev/nvme0n1  1.8T  Samsung SSD 990 PRO
    /dev/sda      465G  Crucial MX500

  Filesystem
  ▸ ext4
    btrfs
    xfs
```

| Filesystem | Choose it if… |
|---|---|
| **ext4** | You want the simple, proven default |
| **btrfs** | You want filesystem snapshots on top of NixOS's own rollbacks |
| **xfs** | You build a lot of code: copy-on-write cloning makes copy-heavy builds such as Rust fast |

Omnix uses the whole disk. Installing next to another operating system is planned for a later version.

### 3. Encrypt the disk?

```text
  Encrypt the disk (LUKS)?
    Yes   ▸ No
```

Choose **Yes** for full-disk encryption with LUKS. You'll type a passphrase at every boot, and the disk is unreadable
without it. Recommended for laptops.

### 4. Choose your system

```text
  What kind of system do you want?
  ▸ Autarchy (stable)  — Omarchy-style keyboard-driven Hyprland, pinned to an Omarchy release
    Autarchy (latest)  — Omarchy-style keyboard-driven Hyprland, tracking Omarchy main
    Atrium             — KDE Plasma desktop: polished windows, mouse-first
    Minimal            — The Omnix base alone: console, FHS layer, nothing else
```

<table>
<tr>
<td width="50%" valign="top">

#### 🟣 Autarchy
**Omarchy, ported to Omnix.**

The keyboard-driven [Omarchy](https://omarchy.org) desktop on Hyprland: tiling windows, Omarchy's keybindings,
themes, and tools, rebuilt as declarative NixOS modules.

- **stable** follows a pinned Omarchy release
- **latest** tracks Omarchy's main branch

</td>
<td width="50%" valign="top">

#### 🪟 Atrium
**Lots of windows, done well.**

A polished, mouse-first KDE Plasma 6 desktop for people who like a taskbar and overlapping windows.

- Built for multiple monitors: each screen has its own virtual desktops and switches them independently
- A slim top bar and a floating dock, set up on first login
- Electron apps, AppImages, and vendor tools just run

</td>
</tr>
</table>

**More styles are coming.** The menu is read live from [`flavors.json`](https://github.com/Omnix-Linux/Omnix/blob/main/flavors.json),
so new flavors appear in the installer without a new ISO. **Minimal** installs just the base, for servers or for
building your own setup.

After choosing, you set a username, password, hostname, and timezone, then confirm one last time before the disk is
erased.

### 5. Sync from the NixOS cache

The installer partitions and formats the disk, writes your machine flake, and installs. Every package is
**downloaded prebuilt from [cache.nixos.org](https://cache.nixos.org)**, the official NixOS binary cache, so nothing
is compiled on your machine, and you get exactly the package versions the ISO was built and tested with.

It also detects your hardware: an NVIDIA GPU gets the NVIDIA driver automatically.

When it finishes, reboot into your new system.

> [!TIP]
> **Changed your mind later?** Your flavor is a single input in `/etc/nixos/flake.nix`. Change it and run
> `nixos-rebuild switch` to switch from Atrium to Autarchy, or the other way round. The previous system stays in
> the boot menu, so you can always go back.

---

## Repo layout

Omnix is deliberately small. The base only provides FHS compatibility. Everything opinionated lives in a
**flavor**, and each flavor is its own repository.

Every install produces a flake that **you** own, in `/etc/nixos`. It stacks four layers, and dependencies only point down:

| Layer | What it is | Lives in |
|---|---|---|
| **Machine** | Your user, hardware, disk, and local tweaks | Your `/etc/nixos` flake |
| **Flavor** | A complete system: desktop, apps, settings | A flavor repo |
| **Base** | NixOS plus the FHS layer | [`Omnix`](https://github.com/Omnix-Linux/Omnix) |
| **nixpkgs** | `nixos-unstable`, unmodified | [cache.nixos.org](https://cache.nixos.org) |

### Repositories

| Repository | Role |
|---|---|
| 🧊 **[`Omnix`](https://github.com/Omnix-Linux/Omnix)** | **The base and the installer.** The FHS module, the installer ISO, the flavor registry (`flavors.json`), and the design docs |
| 🟣 **[`Autarchy`](https://github.com/Omnix-Linux/Autarchy)** | **Flavor: Omarchy, ported.** A keyboard-driven Hyprland desktop, in *stable* (a pinned Omarchy release) and *latest* (Omarchy's main branch) variants |
| 🪟 **[`Atrium`](https://github.com/Omnix-Linux/Atrium)** | **Flavor: KDE Plasma.** A polished, mouse-first Plasma 6 desktop, fully declarative |
| ⚙️ **[`.github`](https://github.com/Omnix-Linux/.github)** | This org profile |

A **Minimal** option, the base alone, is built into the installer.

---

## How we test

- **The base** boots in a NixOS VM under `nix flake check` and runs a collection of real downloaded binaries against
  the FHS layer.
- **The installer** is tested end to end: `tests/install-qemu.py` installs the ISO unattended in QEMU and boots the
  result.
- **Each flavor** has its own VM test that boots its desktop session.
- **Every flavor must be cache-clean:** a dry-run build of a reference machine must show nothing to compile, so
  installing never builds software locally.

Autarchy doubles as the hardest test of all. Omarchy is a script-driven install that assumes Arch, a standard
Linux layout, and prebuilt binaries. If it runs on Omnix, the FHS layer works.

---

## Get started

**Fresh install:** follow the [install walkthrough](#install-walkthrough).

**Existing NixOS (unstable) machine:** add the base to your flake.

```nix
inputs.omnix = { url = "github:Omnix-Linux/Omnix"; inputs.nixpkgs.follows = "nixpkgs"; };
# modules = [ omnix.nixosModules.default ... ];
```

---

## Contributing

- 🐛 **Found a prebuilt binary that won't run?** Open an issue on [`Omnix`](https://github.com/Omnix-Linux/Omnix/issues)
  with the program and the error. Don't diagnose with `ldd`: it bypasses the loader shim and reports working
  binaries as broken.
- 🎨 **Want to build a flavor?** Read the flavor contract in the
  [design doc](https://github.com/Omnix-Linux/Omnix/blob/main/docs/design.md), then open a pull request that adds it to
  `flavors.json`.

<div align="center">
<br>

<img src="assets/omnix-mark.svg" width="64" alt="Omnix mark">

<sub><b>Omnix</b> · <i>NixOS reproducibility with a standard filesystem layout.</i></sub>

<sub>Omnix is independent and not affiliated with or endorsed by the NixOS Foundation.</sub>

</div>
