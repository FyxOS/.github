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

[**Summary**](#executive-summary) · [**Background**](#background) · [**Why it happens**](#why-this-happens) · [**What breaks**](#what-commonly-breaks) · [**How Omnix fixes it**](#how-omnix-fixes-it) · [**Repositories**](#repositories) · [**Testing**](#how-we-test)

</div>

---

## Executive summary

**NixOS can't run most software built for "normal" Linux.** A program downloaded from a vendor, a pip wheel, an npm
package with a native binary, or an AppImage all expect to find their shared libraries in `/lib` and `/usr/lib`.
On NixOS those directories don't exist, so the program fails before its first line of code runs.

This is not a bug. It's a deliberate trade-off at the centre of NixOS's design, and it buys the reproducibility and
safe rollbacks that people choose NixOS for. But the cost is real: every NixOS user eventually hits it, the existing
workarounds have to be set up app by app, and whole distributions like **[Omarchy](https://omarchy.org)** can't run
on NixOS at all.

**Omnix makes NixOS FHS-compliant.** It generates a standard `/lib`, `/usr/lib`, `/bin`, and dynamic-linker cache
from your NixOS configuration, so unmodified Linux software just runs, while the Nix store stays the single source
of truth and every change can still be rolled back.

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

### What the failure looks like

On a stock NixOS install, a downloaded binary is refused at launch:

```console
$ ./some-vendor-tool
Could not start dynamically linked executable: ./some-vendor-tool
NixOS cannot run dynamically linked executables intended for generic
linux environments out of the box. For more information, see:
https://nix.dev/permalink/stub-ld
```

With the common [`nix-ld`](https://github.com/nix-community/nix-ld) workaround enabled, the loader starts, but every
library the program needs has to have been listed by hand in your configuration. Miss one, and you get:

```console
$ ./some-vendor-tool
./some-vendor-tool: error while loading shared libraries: libstdc++.so.6: cannot open shared object file: No such file or directory
```

Scripts fail too, for the same reason: on stock NixOS, `/bin/sh` and `/usr/bin/env` are the only programs at
standard paths, so `#!/bin/bash` and `#!/usr/bin/python3` scripts stop with `bad interpreter: No such file or directory`.

### Why the existing workarounds aren't enough

| Workaround | What it does | The catch |
|---|---|---|
| `patchelf` / `autoPatchelfHook` | Rewrites a binary to point at store paths | Must be repeated for every binary, every update. Breaks signed binaries and tools that verify their own checksums |
| `buildFHSEnv` / `steam-run` | Runs a program inside a namespace that fakes an FHS layout | Per-app wrappers. Programs inside can't see the real system the same way, and setuid helpers and some sandboxes break |
| `nix-ld` | Puts a shim loader at `/lib64/ld-linux-x86-64.so.2` | You still list every library by hand. Doesn't help scripts or anything that hard-codes `/usr/lib` paths |
| `envfs` | Fakes `/bin` and `/usr/bin` for scripts | Covers executables only, not libraries |
| Containers / Distrobox | Runs another distro alongside NixOS | Two systems to maintain, and the software is outside your NixOS config and rollbacks |

Each one fixes part of the problem for one app at a time. None of them makes NixOS look like a normal Linux system
to software that doesn't know it's on NixOS.

---

## What commonly breaks

If it wasn't built by Nix, assume it's affected. The cases NixOS users hit most often:

| Category | Examples | Typical failure |
|---|---|---|
| 🐍 **Python wheels** | NumPy, PyTorch, OpenCV, anything `pip install`ed with compiled parts | `ImportError: libstdc++.so.6` or `libz.so.1: cannot open shared object file` |
| 📦 **npm and other packages that bundle binaries** | esbuild, Prisma engines, Playwright and Puppeteer browsers, sharp | Install succeeds, first run fails on the loader or a missing library |
| 🧑‍💻 **Editor and IDE downloads** | VS Code Remote server, JetBrains plugins, Neovim's Mason language servers | The editor fetches a binary at runtime that can't start |
| 🛠 **Toolchains that download themselves** | `rustup` toolchains, Android SDK and NDK, Arduino and PlatformIO, Bazel, vendor SDKs | Compilers and helper tools fail mid-build |
| 💿 **Vendor apps and AppImages** | Proprietary tarballs, AppImages, closed-source CLIs | Refused at launch by the stub loader |
| 🎮 **Games and GPU workloads** | Native Linux games, CUDA apps, anything that loads `libGL` or `libvulkan` itself | Can't find the graphics driver libraries |
| 📜 **Shell scripts and installers** | `curl … \| sh` installers, scripts that write to `/usr/local` or `/opt` | `#!/bin/bash: bad interpreter`, or writes fail on read-only paths |
| 🐧 **Entire distribution userlands** | **Omarchy**, whose scripts and dotfiles assume an Arch-style FHS system | Doesn't run on NixOS at all today |

---

## How Omnix fixes it

Omnix adds an **FHS layer** to NixOS. On every `nixos-rebuild switch`, it generates the standard Linux layout from
your configuration:

- **`/lib64/ld-linux-x86-64.so.2`** is a real loader, so unmodified binaries start.
- **`/lib`, `/usr/lib`, and `/etc/ld.so.cache`** are populated from the libraries in your system configuration, so
  the loader finds them the normal way, with no hand-maintained list.
- **`/bin`, `/usr/bin`, and `/usr/share`** are populated, so `#!/bin/bash`-style scripts and data lookups work.
- **`/usr/local` and `/opt`** are writable, so ordinary installers have somewhere to install.

The layer is generated from the Nix store, never edited by hand, and versioned with the rest of the system. Roll
back a NixOS generation and the FHS layer rolls back with it. On top of that, **distro profiles** such as Omarchy
run their upstream code on NixOS, unmodified.

---

## Repositories

| Repository | Role |
|---|---|
| 🧊 **[`omnix`](https://github.com/Omnix-Linux/omnix)** | **Core flake.** NixOS modules, system templates, and hardware presets |
| 📂 **[`omnix-fhs`](https://github.com/Omnix-Linux/omnix-fhs)** | **The FHS layer.** Generates the library and executable directories, the loader cache, and the writable overlays |
| 🟣 **[`omnix-omarchy`](https://github.com/Omnix-Linux/omnix-omarchy)** | **Omarchy profile.** Runs upstream Omarchy and its Hyprland desktop on Omnix |
| 🐧 **[`omnix-distros`](https://github.com/Omnix-Linux/omnix-distros)** | **Other distro profiles.** Arch, Debian-style, and minimal userlands |
| 💿 **[`omnix-iso`](https://github.com/Omnix-Linux/omnix-iso)** | **Installer media.** Live ISO with a guided installer |
| 🧪 **[`omnix-tests`](https://github.com/Omnix-Linux/omnix-tests)** | **Integration tests.** VM tests, FHS conformance, and screenshot tests |
| 📚 **[`omnix-docs`](https://github.com/Omnix-Linux/omnix-docs)** | **Docs and website** |

---

## How we test

The FHS layer sits under every program on the system, so a regression there breaks everything at once. Every pull
request must pass five checks, each in a disposable VM:

| Check | What it proves | How |
|---|---|---|
| **1. Evaluate** | Every module and profile evaluates and is lint-clean | `nix flake check`, `nixfmt`, `statix`, `deadnix` |
| **2. Unit** | The FHS generator is deterministic and correct | Snapshot and property tests on the generated directories and loader cache |
| **3. FHS conformance** | Unmodified Linux software can't tell it isn't on a normal distro | Path audit against FHS 3.0, a collection of **unpatched** real-world binaries (glibc, musl, 32-bit), pip wheels, and a shebang matrix |
| **4. Distro VM tests** | Omarchy and the other profiles work from first boot to desktop | NixOS VM tests install each profile, log in to Hyprland, and check the screen with OCR and screenshot comparison |
| **5. Resilience** | Upgrades never leave a machine unbootable | Upgrade, roll back, and deliberately break an activation that must revert automatically |

Every check also runs locally with `nix flake check` and `nix run .#vm-test -- <profile>`.

---

## Quick start

```nix
{
  inputs.omnix.url = "github:Omnix-Linux/omnix";

  outputs = { nixpkgs, omnix, ... }: {
    nixosConfigurations.my-machine = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        omnix.nixosModules.default
        {
          omnix.fhs.enable = true;          # real /lib, /usr/lib, /usr/bin, …
          omnix.profile    = "omarchy";     # or "arch", "minimal", …
        }
        ./hardware-configuration.nix
      ];
    };
  };
}
```

Then run `sudo nixos-rebuild switch --flake .#my-machine`.

---

## Contributing

- 🐛 **Found software that won't run?** Open an issue with the program, the error, and `ldd <binary>` output. We'll
  add it to the conformance tests.
- 🐧 **Want your distribution supported?** Profiles live in [`omnix-distros`](https://github.com/Omnix-Linux/omnix-distros).
- 🧪 **Want to strengthen testing?** [`omnix-tests`](https://github.com/Omnix-Linux/omnix-tests) always needs more
  real-world binaries and screenshot baselines.

<div align="center">
<br>

<img src="assets/omnix-mark.svg" width="64" alt="Omnix mark">

<sub><b>Omnix</b> · <i>NixOS reproducibility with a standard filesystem layout.</i></sub>

</div>
