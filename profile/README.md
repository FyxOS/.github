<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/fyxos-banner-dark.svg">
  <source media="(prefers-color-scheme: light)" srcset="assets/fyxos-banner-light.svg">
  <img alt="fyxos — FHS-compliant NixOS" src="assets/fyxos-banner-light.svg" width="720">
</picture>

### NixOS reproducibility, with an ordinary `/usr/bin`.

**fyxos** is NixOS made **FHS-compliant**, so [Omarchy](https://omarchy.org) and other distributions' userlands
run on a declarative, rollback-safe base, without patchelf or wrapper scripts.

<br>

[![NixOS](https://img.shields.io/badge/NixOS-unstable-5277C3?style=for-the-badge&logo=nixos&logoColor=white)](https://nixos.org)
[![FHS 3.0](https://img.shields.io/badge/FHS-3.0_compliant-7EBAE4?style=for-the-badge&logo=linuxfoundation&logoColor=white)](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html)
[![Omarchy](https://img.shields.io/badge/Runs-Omarchy-8B5CF6?style=for-the-badge&logo=archlinux&logoColor=white)](https://omarchy.org)
[![Hyprland](https://img.shields.io/badge/Wayland-Hyprland-58E1FF?style=for-the-badge&logo=wayland&logoColor=black)](https://hyprland.org)

[**Why**](#-why-fyxos) · [**Architecture**](#-architecture) · [**Repositories**](#-the-org-at-a-glance) · [**Testing**](#-how-we-test) · [**Quick start**](#-quick-start) · [**Contributing**](#-contributing)

</div>

---

## ✨ Why fyxos?

NixOS is excellent at reproducibility, but software written for "normal" Linux breaks on it. Prebuilt binaries look for
`/lib64/ld-linux-x86-64.so.2`, scripts start with `#!/bin/bash`, and installers write to `/usr/local`, `/opt`, and `/etc`.
On stock NixOS none of those paths behave the way the software expects.

Opinionated distributions like **Omarchy** depend on those paths: they ship shell scripts, dotfiles, and binaries that assume
a standard Filesystem Hierarchy. fyxos provides that hierarchy while keeping what makes NixOS useful.

<table>
<tr>
<td width="50%" valign="top">

#### 🧊 What you keep from NixOS
- One `flake.nix` describes the whole machine
- Atomic upgrades with **boot-menu rollbacks**
- Bit-for-bit reproducible system closures
- The `nixpkgs` package set, with 100k+ packages

</td>
<td width="50%" valign="top">

#### 📂 What fyxos adds
- A real `/bin`, `/usr/bin`, `/lib`, `/lib64`, `/usr/lib`, `/usr/share`
- A working dynamic loader, so **unpatched ELF binaries just run**
- `#!/bin/bash` and other FHS shebangs resolve normally
- **Distro profiles** that run Omarchy (and others) unmodified

</td>
</tr>
</table>

---

## 🏗 Architecture

fyxos is built in layers. The Nix store is still the single source of truth. The FHS layer is a **generated,
read-only projection** of the store onto standard paths, rebuilt on every `nixos-rebuild switch`.

```mermaid
flowchart TB
    subgraph USER["👤 User space"]
        direction LR
        OM["🟣 Omarchy<br/>Hyprland · scripts · dotfiles"]
        OD["🐧 Other distro profiles<br/>Arch · Debian-style userlands"]
        BIN["📦 Unpatched binaries<br/>AppImages · vendor tarballs · games"]
    end

    subgraph FHS["📂 fyxos FHS layer"]
        direction LR
        PATHS["/bin · /usr/bin · /sbin<br/>/lib · /lib64 · /usr/lib"]
        LD["Dynamic loader shim<br/>ld-linux + ld.so.cache"]
        ETC["/etc · /opt · /usr/local<br/>mutable overlays"]
    end

    subgraph NIX["❄️ NixOS core"]
        direction LR
        MOD["fyxos NixOS modules"]
        STORE[("/nix/store")]
        GEN["System generations<br/>+ rollback"]
    end

    KERNEL["🐧 Linux kernel"]

    USER --> FHS
    FHS -- "symlink farm & loader cache<br/>generated at activation" --> NIX
    MOD --> STORE
    STORE --> GEN
    NIX --> KERNEL

    classDef user fill:#2E1065,stroke:#8B5CF6,color:#F5F3FF
    classDef fhs fill:#0C2A4A,stroke:#7EBAE4,color:#E6EDF7
    classDef nix fill:#13254D,stroke:#5277C3,color:#E6EDF7
    classDef k fill:#111827,stroke:#6B7280,color:#E5E7EB
    class OM,OD,BIN user
    class PATHS,LD,ETC fhs
    class MOD,STORE,GEN nix
    class KERNEL k
```

### How a foreign binary finds its libraries

```mermaid
sequenceDiagram
    autonumber
    participant App as 📦 Prebuilt app
    participant K as 🐧 Kernel
    participant LD as /lib64/ld-linux-x86-64.so.2
    participant C as ld.so.cache
    participant S as /nix/store

    App->>K: execve("/usr/bin/app")
    K->>LD: Load interpreter from ELF PT_INTERP
    Note over LD: fyxos shim. On stock NixOS<br/>this path does not exist.
    LD->>C: Resolve libc.so.6, libGL.so.1, …
    C-->>LD: /usr/lib/libGL.so.1 → /nix/store/…-mesa/lib
    LD->>S: mmap the real libraries
    S-->>App: ✅ Runs unmodified, with no patchelf
```

### Activation lifecycle

```mermaid
flowchart LR
    A["✍️ Edit flake.nix"] --> B["nixos-rebuild switch"]
    B --> C["Build closure<br/>in /nix/store"]
    C --> D["fyxos activation hook"]
    D --> E["Regenerate FHS<br/>symlink farm"]
    D --> F["Rebuild<br/>ld.so.cache"]
    D --> G["Reconcile /etc<br/>overlays"]
    E & F & G --> H{"FHS self-check<br/>passes?"}
    H -- yes --> I["🎉 New generation live"]
    H -- no --> J["⏪ Auto-rollback to<br/>previous generation"]

    style I fill:#14532D,stroke:#22C55E,color:#F0FDF4
    style J fill:#7F1D1D,stroke:#EF4444,color:#FEF2F2
```

---

## 🗺 The org at a glance

| Repository | Role | Highlights |
|---|---|---|
| 🧊 **[`fyxos`](https://github.com/FyxOS/fyxos)** | **Core flake.** The entry point, NixOS modules, and system templates | `nixosModules.default`, `templates.*`, hardware presets |
| 📂 **[`fyxos-fhs`](https://github.com/FyxOS/fyxos-fhs)** | **The FHS layer.** Symlink-farm generator, loader shim, `/etc` overlay reconciler | Activation hooks, `ld.so.cache` builder, FHS self-check |
| 🟣 **[`fyxos-omarchy`](https://github.com/FyxOS/fyxos-omarchy)** | **Omarchy profile.** Runs upstream Omarchy on fyxos | Hyprland session, pacman/yay shims, theme sync |
| 🐧 **[`fyxos-distros`](https://github.com/FyxOS/fyxos-distros)** | **Other distro profiles.** Arch, Debian-style, and minimal userlands | Profile schema, community-contributed profiles |
| 💿 **[`fyxos-iso`](https://github.com/FyxOS/fyxos-iso)** | **Installer media.** Live ISO with a guided installer | Graphical and TUI installers, disko layouts |
| 🧪 **[`fyxos-tests`](https://github.com/FyxOS/fyxos-tests)** | **Integration test suite.** VM tests, conformance, and screenshot tests | `nixosTest` matrix, FHS 3.0 checker, Hyprland visual tests |
| 📚 **[`fyxos.dev`](https://github.com/FyxOS/fyxos.dev)** | **Docs and website** | Guides, architecture notes, profile authoring |
| ⚙️ **[`.github`](https://github.com/FyxOS/.github)** | Org profile, community health files, issue templates | You are here 👋 |

```mermaid
flowchart LR
    FHS["📂 fyxos-fhs"] --> CORE["🧊 fyxos"]
    CORE --> OMA["🟣 fyxos-omarchy"]
    CORE --> DIS["🐧 fyxos-distros"]
    OMA --> ISO["💿 fyxos-iso"]
    DIS --> ISO
    TESTS["🧪 fyxos-tests<br/><i>gates every PR</i>"] -.-> FHS & CORE & OMA & DIS & ISO
    DOCS["📚 fyxos.dev"] -.-> CORE

    classDef core fill:#13254D,stroke:#5277C3,color:#E6EDF7
    classDef prof fill:#2E1065,stroke:#8B5CF6,color:#F5F3FF
    classDef qa fill:#14532D,stroke:#22C55E,color:#F0FDF4
    class FHS,CORE,ISO core
    class OMA,DIS prof
    class TESTS,DOCS qa
```

---

## 🧪 How we test

The FHS layer sits under every program on the system, so a regression there breaks everything at once.
Every change goes through **five gates** before it reaches a user, and each gate runs in a disposable VM.

```mermaid
flowchart LR
    PR(["🔀 Pull request"]) --> L1

    subgraph L1["① Eval"]
        E1["nix flake check"]
        E2["nixfmt · statix · deadnix"]
    end

    subgraph L2["② Unit"]
        U1["Symlink-farm generator<br/>golden-file tests"]
        U2["ld.so.cache builder<br/>property tests"]
    end

    subgraph L3["③ FHS conformance"]
        C1["FHS 3.0 path audit"]
        C2["Unpatched ELF corpus<br/>glibc · musl · 32-bit"]
        C3["Shebang matrix<br/>/bin/sh · /usr/bin/env · bash"]
    end

    subgraph L4["④ Distro VM tests"]
        V1["Omarchy install<br/>end-to-end"]
        V2["Hyprland session<br/>OCR + screenshot diff"]
        V3["Other distro profiles"]
    end

    subgraph L5["⑤ Resilience"]
        R1["Upgrade A → B"]
        R2["Rollback B → A"]
        R3["Broken-activation<br/>auto-rollback"]
    end

    L1 --> L2 --> L3 --> L4 --> L5 --> M(["✅ Merge"])

    style PR fill:#13254D,stroke:#5277C3,color:#E6EDF7
    style M fill:#14532D,stroke:#22C55E,color:#F0FDF4
```

| Gate | What it proves | How |
|---|---|---|
| **① Eval** | Every module, profile, and host config evaluates and is lint-clean | `nix flake check`, `nixfmt`, `statix`, `deadnix` |
| **② Unit** | The FHS generators are deterministic and correct | Golden-file snapshots and property tests on the symlink farm and loader cache |
| **③ FHS conformance** | A standard Linux program can't tell it isn't on a normal distro | Path audit against **FHS 3.0**, plus a corpus of **unpatched** vendor binaries (glibc, musl, i686) and a shebang matrix |
| **④ Distro VM tests** | Omarchy and the other profiles work from first boot to desktop | `nixosTest` VMs install each profile, log in to Hyprland, and assert on **OCR + screenshot diffs** |
| **⑤ Resilience** | Upgrades never leave a machine unbootable | Upgrade, rollback, and deliberately broken activations that must auto-revert |

> [!TIP]
> Every gate runs locally too. You don't have to push to find out whether CI will pass:
> ```bash
> nix flake check                      # gates ① + ②
> nix build .#checks.x86_64-linux.fhs  # gate ③
> nix run   .#vm-test -- omarchy       # gate ④, opens the VM interactively
> ```

### Test matrix

| | `x86_64-linux` | `aarch64-linux` |
|---|:---:|:---:|
| **FHS conformance** | ✅ | ✅ |
| **Unpatched ELF corpus** | ✅ | ✅ |
| **Omarchy profile** | ✅ | 🚧 |
| **Other distro profiles** | ✅ | 🚧 |
| **Installer ISO** | ✅ | 🚧 |

<sub>✅ gated on every PR · 🚧 in progress</sub>

---

## 🚀 Quick start

**Add fyxos to an existing NixOS flake:**

```nix
{
  inputs.fyxos.url = "github:fyxos/fyxos";

  outputs = { nixpkgs, fyxos, ... }: {
    nixosConfigurations.my-machine = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        fyxos.nixosModules.default
        {
          fyxos.fhs.enable = true;          # real /usr/bin, /lib64, …
          fyxos.profile    = "omarchy";     # or "arch", "minimal", …
        }
        ./hardware-configuration.nix
      ];
    };
  };
}
```

**Or start from a template:**

```bash
nix flake init -t github:fyxos/fyxos#omarchy
sudo nixos-rebuild switch --flake .#my-machine
```

**Check it worked:**

```console
$ ls /usr/bin/bash /lib64/ld-linux-x86-64.so.2
/usr/bin/bash  /lib64/ld-linux-x86-64.so.2
$ fyxos doctor
✔ FHS layer       generation 42, 18,311 paths projected
✔ Dynamic loader  ld.so.cache fresh
✔ Profile         omarchy (Hyprland session ready)
```

---

## 🤝 Contributing

Contributions are welcome. Good places to start:

- 🐛 **Found a binary that won't run?** Open an issue with the output of `fyxos doctor` and `ldd <binary>`, and we'll add it to the conformance corpus.
- 🐧 **Want your favorite distro supported?** Profiles live in [`fyxos-distros`](https://github.com/FyxOS/fyxos-distros). Copy an existing one to start.
- 🧪 **Want to strengthen testing?** [`fyxos-tests`](https://github.com/FyxOS/fyxos-tests) always needs more real-world binaries and screenshot baselines.

Every PR must pass all five test gates. Run them locally before pushing.

<div align="center">
<br>

<img src="assets/fyxos-mark.svg" width="64" alt="fyxos mark">

<sub><b>fyxos</b> · <i>NixOS reproducibility with a standard filesystem layout.</i></sub>

</div>
