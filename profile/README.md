<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/omnix-banner-dark.svg">
  <source media="(prefers-color-scheme: light)" srcset="assets/omnix-banner-light.svg">
  <img alt="Omnix — FHS-compliant NixOS" src="assets/omnix-banner-light.svg" width="720">
</picture>

### NixOS reproducibility, with an ordinary `/usr/bin`.

[![NixOS](https://img.shields.io/badge/NixOS-unstable-5277C3?style=for-the-badge&logo=nixos&logoColor=white)](https://nixos.org)
[![FHS 3.0](https://img.shields.io/badge/FHS-3.0_compliant-7EBAE4?style=for-the-badge&logo=linuxfoundation&logoColor=white)](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html)
[![Omarchy](https://img.shields.io/badge/Runs-Omarchy-8B5CF6?style=for-the-badge&logo=archlinux&logoColor=white)](https://omarchy.org)

[**Summary**](#executive-summary) · [**Background**](#background) · [**Why it happens**](#why-this-happens) · [**How Omnix fixes it**](#how-omnix-fixes-it) · [**Repo layout**](#repo-layout) · [**Testing**](#how-we-test) · [**Get started**](#get-started)

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

## Repo layout

Omnix is deliberately small. The base only provides FHS compatibility. Everything opinionated lives in a
**flavor**, and each flavor is its own repository.

### From boot to a running system

```text
 1. Boot the Omnix installer      a small ISO: stock NixOS installer + the Omnix base
          │
 2. Bring up the network          wired DHCP, or nmtui for Wi-Fi
          │
 3. Choose a flavor               read live from flavors.json, so a new flavor needs no new ISO
          │
 4. Write your machine flake      Omnix base + the chosen flavor + your hardware and user
          │
 5. Sync and build                download everything from cache.nixos.org; nothing is compiled
          │
 6. Reboot into your system       later, switch flavors by changing one flake input and rebuilding
```

The result is a flake that **you** own, in `/etc/nixos`. It stacks four layers, and dependencies only point down:

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

**Fresh install:** download the installer ISO from [Releases](https://github.com/Omnix-Linux/Omnix/releases),
boot it, and run the installer. It walks you through the network, disk, flavor, and user.

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
