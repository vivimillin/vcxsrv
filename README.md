# VcXsrv — with WSL2 vsock & Relative Mouse Mode (Unofficial Builds)

[![Release](https://img.shields.io/github/v/release/vivimillin/vcxsrv)](../../releases)
[![License](https://img.shields.io/badge/license-X11-blue)](COPYING)

**English** · [中文版](README.zh.md)

Unofficial builds of [VcXsrv](https://github.com/marchaesen/vcxsrv) — the open-source X server for Windows — with two patch sets that close the gap between VcXsrv and a first-class display server for Linux GUI apps on WSL: the WSL2 transport and mouse capture.

- **Hyper-V vsock transport for WSL2** — a direct VM↔host channel that replaces the TCP / localhost-forwarding path: lower latency under input-heavy load, unaffected by VPN switching or sleep/wake, and effectively zero configuration. **Display responsiveness on par with WSLg and the commercial X410 for 2D desktop workloads (WSLg keeps the edge in GPU-accelerated 3D/video) — a fully open-source X410 alternative for WSL2.**

- **Relative mouse mode & cursor confinement** — hardware-level `XI_RawMotion` plus proper `XGrabPointer` confinement and hiding, **so SDL/SDL2 games, emulators, and 3D/CAD tools capture the mouse correctly**. Broken relative mouse mode is a long-standing WSL pain point — including on WSLg ([microsoft/wslg#240](https://github.com/microsoft/wslg/issues/240), [#521](https://github.com/microsoft/wslg/issues/521)) and X410.

All patches are submitted upstream as proper PRs to [marchaesen/vcxsrv](https://github.com/marchaesen/vcxsrv) (PR #78; Issue #80 with a PR to follow). Upstream releases roughly once a year, so this project provides builds in the meantime — each patch is retired once it lands upstream.

[Releases](../../releases) · [Upstream PR #78](https://github.com/marchaesen/vcxsrv/pull/78) · [Issue #77](https://github.com/marchaesen/vcxsrv/issues/77) · [Issue #80](https://github.com/marchaesen/vcxsrv/issues/80)

---

## Who Is This For

**Anyone running Linux GUI apps on WSL2 — the vsock transport alone justifies these builds.** Upstream VcXsrv reaches WSL2 over TCP (NAT + localhost forwarding) — the path behind well-known annoyances: stutter under mouse-dense workloads (dragging, scrolling, SDL2 event floods), connections severed or hung after sleep/wake or Wi-Fi/VPN changes. `-wslvsock` moves X11 traffic to a Hyper-V socket — a channel purpose-built for VM↔host communication that bypasses the TCP/IP stack entirely:

- **Performance** — no NAT, no localhost proxy, no network-stack overhead in the data path; input-heavy workloads stop stuttering
- **Stability** — unaffected by VPN changes, Wi-Fi roaming, adapter power saving, or sleep/wake
- **Zero configuration, zero privileges** — the running WSL2 VM is auto-detected at startup and re-detected after every WSL restart; no `DISPLAY` IP juggling, no admin rights, no "Hyper-V Administrators" group membership — which [X410's reliable WSL2 detection requires](https://x410.dev/cookbook/wsl/using-x410-with-wsl2/)

**Anyone running mouse-capturing X11 applications** — on WSL1/WSL2 or any setup using VcXsrv as the display server — gets two X11-protocol-level fixes in the Windows DDX layer (`hw/xwin`):

- **SDL2 games** requesting relative mouse mode (dosbox-staging, AssaultCube, crispy-doom, Quake-based engines…)
- **3D/CAD tools** capturing the mouse for viewport navigation (Blender, FreeCAD)
- **Remote desktop / VNC viewers and emulators** that lock the local cursor — any X11 client calling `XGrabPointer` with a confine window

## How It Compares

| | WSLg | X410 | VcXsrv (upstream) | **These builds** |
| --- | --- | --- | --- | --- |
| Open source | Linux-side components | ✗ (proprietary, paid) | ✓ | ✓ |
| WSL2 transport | local socket (fast) | Hyper-V vsock (fast) | TCP via NAT / localhost forwarding | **Hyper-V vsock (fast)** |
| WSL2 setup | none | minimal; reliable vsock detection needs the 'Hyper-V Administrators' group² | manual `DISPLAY` / host IP | one flag + one `socat` line, no admin |
| Relative mouse mode (`XI_RawMotion`) | broken for many games¹ | user-reported issues; no public tracker | never generated | ✓ working |
| Cursor confinement & hiding on grab | broken for many games¹ | user-reported issues; no public tracker | not implemented | ✓ working |

> ¹ Publicly tracked: [microsoft/wslg#240](https://github.com/microsoft/wslg/issues/240) ("mouselock isnt working and mouse input is chaotic in video games"), [microsoft/wslg#521](https://github.com/microsoft/wslg/issues/521) ("Games can't catch cursor").<br>
> ² Per the [X410 documentation](https://x410.dev/cookbook/wsl/using-x410-with-wsl2/), the codes that "more reliably detect WSL2" for vsock "require additional user privileges for accessing Hyper-V related API's in Windows" (membership in the 'Hyper-V Administrators' group). This project detects the VM via `wsl.exe -- wslinfo --vm-id` instead, which needs no elevation at all.

Nothing here replaces what WSLg or X410 already do well — the point is that a **fully open-source** option no longer has to accept a slower transport or broken mouse capture as the price of entry.

---

## What This Provides

| Patch set | What it fixes | Upstream status | Notes |
| --- | --- | --- | --- |
| **Raw Input mouse** | XInput2 `XI_RawMotion` never generated; SDL2 relative mouse mode broken | [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) — Open | [Implementation Notes.1](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-SDL2-Relative-Mouse-Mode-Fix) |
| **Cursor confinement & hiding** | `XGrabPointer` with `confineTo` has no effect; cursor stays visible and free | Same [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) — Open | [Implementation Notes.1](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-SDL2-Relative-Mouse-Mode-Fix) |
| **Empty-mask cursor hiding** | `XDefineCursor` with an all-zero-mask cursor left the previous cursor image on screen (cursor never hidden) | Same [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) — Open | [Implementation Notes.2](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-Empty-Cursor-Hide-Fix) |
| **WSL2 vsock transport** | zero-config listener for WSL2; upstream hyperv listener never matches WSL2's VM; display number ignored on bind | [Issue #80](https://github.com/marchaesen/vcxsrv/issues/80) — Open, PR to follow | [Implementation Notes.3](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-WSL2-vsock-Fix) |

> Unofficial builds are on the [Releases page](../../releases); each release lists exactly which patches it contains. In-depth design and implementation write-ups for each patch set are in the wiki, linked in the **Notes** column above.

---

## Quick Start

1. **Install** the latest build from the [Releases page](../../releases).

2. **Start the server (Windows side):**
   ```bash
   # WSL2 - recommended: vsock transport & mouse patches
   vcxsrv.exe :0 -multiwindow -clipboard -wgl -wslvsock

   # WSL1 or any non-WSL setup - only mouse patches, need no extra configuration
   vcxsrv.exe -multiwindow -clipboard -wgl
   ```
   > XLaunch users: equivalently set `ExtraParams="-wslvsock"` in `config.xlaunch`
   > (the wizard's "Additional parameters for VcXsrv" field).

3. **Connect (WSL2 side)** - one socat forwarder (requires Store WSL 2.0+ and socat ≥ 1.7.4):
   ```bash
   socat UNIX-LISTEN:/tmp/.X11-unix/X0,fork,mode=777,forever,retry=10,interval=2 VSOCK-CONNECT:2:106000 &
   export DISPLAY=:0
   ```
   > Add both lines to ~/.bashrc so every new shell works out of the box.

4. **Done.** For autostart at login, WSLg coexistence, and troubleshooting, see the [https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-WSL2-vsock-User-Guide](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-WSL2-vsock-User-Guide).
   > **Tip:** When running DOSBox-Staging on WSL, if the cursor stays still in seamless mode (does not follow the system cursor), set `export XDG_CURRENT_DESKTOP=WSL` before launching. SDL2 uses this to decide whether to enable X11 mouse integration.

---

## Tested With

Mouse patches:

| Application | Platform | Mode | Result |
| --- | --- | --- | --- |
| dosbox-staging 0.82 | WSL1/WSL2 | seamless + capture | Pass |
| AssaultCube 1.3 | WSL1/WSL2 | capture + release | Pass |
| crispy-doom 5.11 | WSL1/WSL2 | capture + release | Pass |

vsock transport (Windows 11 + Store WSL2, Ubuntu):

| Validation | Result |
| --- | --- |
| X11 handshake over vsock | `SUCCESS`, cookie-free (FamilyLocal) |
| Dual instances `:11` / `:12` with `-wslvsock` | both connect on 106011 / 106012 (display-offset fix) |
| WSL VM restarted while server keeps running | listener auto-rebound to new VM id; clients reconnect |
| Default start (no `-wslvsock`) | no vsock listener; TCP path unchanged (no regression) |

---

## Help These Patches Land Upstream

The most useful thing you can do — more than starring this repo — is to test the upstream submissions and leave your results there:

- [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) (mouse patches) and [Issue #77](https://github.com/marchaesen/vcxsrv/issues/77)
- [Issue #80](https://github.com/marchaesen/vcxsrv/issues/80) (vsock transport)

Confirmed test reports from real users are what moves long-running PRs forward, and once merged, everyone gets these fixes in the official builds. Bugs specific to these unofficial builds can be reported in this repo's issue tracker.

> Development, issues, and releases live in this repo; upstream PRs are staged through the shadow fork [vivimillin/vcxsrv-upstream](https://github.com/vivimillin/vcxsrv-upstream).

---

## Roadmap

**Done**
- [x] Raw Input mouse (`WM_INPUT` → `XI_RawMotion`)
- [x] Cursor confinement & hiding on grab
- [x] Empty-mask cursor hiding (SDL seamless mode)
- [x] AF_VSOCK transport — zero-config, self-healing WSL2 VM↔host channel

**In progress / Planned**
- [ ] Upstream merge of PR #78; vsock PR following Issue #80
- [ ] Clipboard improvements — smoother bidirectional text/image sharing between Windows host and X11 clients

---

## Building from Source

- **Clone the branch with the patches you want**

   ```bash
   # WSL2 vsock transport & Mouse patches (Issue #80 & PR #78)
   git clone -b feature/vsock-wsl2 https://github.com/vivimillin/vcxsrv.git

   # Mouse patches only (PR #78)
   git clone -b feature/raw-input-mouse https://github.com/vivimillin/vcxsrv.git
   ```

- **Follow the step-by-step guide:** [Building VcXsrv on WSL](https://github.com/vivimillin/vcxsrv/wiki/Building-VcXsrv-on-WSL)

---

## Branches

| Branch | Purpose | Default? |
| --- | --- | --- |
| `master` | Clean mirror of upstream | No  |
| `pages` | GitHub homepage, this README + upstream source | **Yes** |
| `feature/raw-input-mouse` | PR #78 code | No  |
| `feature/vsock-wsl2` | vsock transport code (Issue #80) | No  |

`master` tracks upstream via `reset --hard` and carries no local commits. All project-specific content lives on `pages`.

---

## License

MIT-style X11 license, same as upstream VcXsrv.

