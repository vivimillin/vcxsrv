# VcXsrv — with WSL2 vsock & Relative Mouse Mode (Unofficial Builds)

**English** · [中文](#中文简介)

Unofficial builds of [VcXsrv](https://github.com/marchaesen/vcxsrv) — the open-source X server for Windows — with two patch sets that close the gap between VcXsrv and a first-class display server for Linux GUI apps on WSL: the WSL2 transport and mouse capture.

- **Hyper-V vsock transport for WSL2** — a direct VM↔host channel that replaces the TCP / localhost-forwarding path: lower latency under input-heavy load, unaffected by VPN switching or sleep/wake, and effectively zero configuration. **Display responsiveness on par with WSLg and the commercial X410 — a fully open-source X410 alternative for WSL2.**

- **Relative mouse mode & cursor confinement** — hardware-level `XI_RawMotion` plus proper `XGrabPointer` confinement and hiding, **so SDL/SDL2 games, emulators, and 3D/CAD tools capture the mouse correctly**. Broken relative mouse mode is a long-standing WSL pain point — including on WSLg ([microsoft/wslg#240](https://github.com/microsoft/wslg/issues/240), [#521](https://github.com/microsoft/wslg/issues/521)) and X410.

All patches are submitted upstream as proper PRs ([PR #78](https://github.com/marchaesen/vcxsrv/pull/78); [Issue #80](https://github.com/marchaesen/vcxsrv/issues/80) with a PR to follow). Upstream releases roughly once a year, so this fork provides builds in the meantime — each patch is retired once it lands upstream.

[Releases](../../releases) · [Upstream PR #78](https://github.com/marchaesen/vcxsrv/pull/78) · [Issue #77](https://github.com/marchaesen/vcxsrv/issues/77) · [Issue #80](https://github.com/marchaesen/vcxsrv/issues/80)

---

## Who Is This For

**Anyone running Linux GUI apps on WSL2 — the vsock transport alone justifies these builds.**

Upstream VcXsrv talks to WSL2 over TCP, through the NAT and localhost-forwarding layers. That path is behind several well-known annoyances: stutter under mouse-dense workloads (dragging, scrolling, SDL2 event floods), connections severed or hung after sleep/wake or Wi-Fi/VPN changes. 

With `-wslvsock`, X11 traffic moves to a Hyper-V socket — a channel purpose-built for VM↔host communication that bypasses the TCP/IP stack entirely:

- **Performance** — no NAT, no localhost proxy, no network-stack overhead in the data path; input-heavy workloads stop stuttering
- **Stability** — unaffected by VPN changes, Wi-Fi roaming, adapter power saving, or sleep/wake
- **Zero configuration, zero privileges** — the running WSL2 VM is detected automatically at startup and re-detected automatically after every WSL restart; no `DISPLAY` IP juggling, no admin rights, and no "Hyper-V Administrators" group membership — which [X410's reliable WSL2 detection requires](https://x410.dev/cookbook/wsl/using-x410-with-wsl2/)

**Anyone running mouse-capturing X11 applications** — on WSL1/WSL2 or any setup using VcXsrv as the display server — gets two X11-protocol-level fixes in the Windows DDX layer (`hw/xwin`):

- **SDL2 applications** requesting relative mouse mode (dosbox-staging, AssaultCube, crispy-doom, Quake-based engines, many other games)
- **3D/CAD tools** that capture the mouse for viewport navigation (Blender, FreeCAD)
- **Remote desktop / VNC viewers** on X11 that lock the local cursor
- **Emulators** using mouse capture for host integration
- Any X11 client calling `XGrabcursor` with a confine window

## How It Compares

| | WSLg | X410 | VcXsrv (upstream) | **This fork** |
| --- | --- | --- | --- | --- |
| Open source | Linux-side components | ✗ (proprietary, paid) | ✓ | ✓ |
| WSL2 transport | local socket (fast) | Hyper-V vsock (fast) | TCP via NAT / localhost forwarding | **Hyper-V vsock (fast)** |
| WSL2 setup | none | minimal; reliable vsock detection needs the 'Hyper-V Administrators' group² | manual `DISPLAY` / host IP | one flag + one `socat` line, no admin |
| Relative mouse mode (`XI_RawMotion`) | broken for many games¹ | user-reported issues; no public tracker | never generated | ✓ working |
| Cursor confinement & hiding on grab | broken for many games¹ | user-reported issues; no public tracker | not implemented | ✓ working |

¹ Publicly tracked: [microsoft/wslg#240](https://github.com/microsoft/wslg/issues/240) ("mouselock isnt working and mouse input is chaotic in video games"), [microsoft/wslg#521](https://github.com/microsoft/wslg/issues/521) ("Games can't catch cursor").

² Per the [X410 documentation](https://x410.dev/cookbook/wsl/using-x410-with-wsl2/), the codes that "more reliably detect WSL2" for vsock "require additional user privileges for accessing Hyper-V related API's in Windows" (membership in the 'Hyper-V Administrators' group). This fork detects the VM via `wsl.exe -- wslinfo --vm-id` instead, which needs no elevation at all.

Nothing here replaces what WSLg or X410 already do well — the point is that a **fully open-source** option no longer has to accept a slower transport or broken mouse capture as the price of entry.

---

## What This Provides

| Patch set | What it fixes | Upstream status |
| --- | --- | --- |
| **WSL2 vsock transport** | hyperv listener never matches WSL2's VM; display number ignored on bind; listener on by default | [Issue #80](https://github.com/marchaesen/vcxsrv/issues/80) — Open, PR to follow |
| **Raw Input mouse** | XInput2 `XI_RawMotion` never generated; SDL2 relative mouse mode broken | [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) — Open |
| **Cursor confinement & hiding** | `XGrabcursor` with `confineTo` has no effect; cursor stays visible and free | Same [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) — Open |
| **Empty-mask cursor hiding** | `XDefineCursor` with an all-zero-mask cursor left the previous cursor image on screen (cursor never hidden) | Same [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) — Open |

Unofficial builds are on the [Releases page](../../releases); each release lists exactly which patches it contains.

### WSL2 vsock transport (`-wslvsock`)

VcXsrv has shipped a Hyper-V vsock transport for years, but it never worked for WSL2 out of the box: the listener binds a wildcard VM id that WSL2's utility VM refuses to match, and the exact VM id it needs changes on every WSL restart and was only discoverable with elevated tools. `-wslvsock` closes that gap: it detects the running WSL2 VM automatically via `wsl.exe -- wslinfo --vm-id` — **no admin rights, no "Hyper-V Administrators" group** (unlike the [HCS-API approach used by X410](https://x410.dev/cookbook/wsl/using-x410-with-wsl2/)) — binds the listener to that VM at port `106000 + display`, and watches the VM instance so the listener rebinds to the new id in ~1–2 s after every WSL restart, without restarting VcXsrv. On WSL1, or with no WSL2 VM present, it silently falls back to TCP. vsock stays off by default (no wildcard listener exposure, no log noise on non-Hyper-V hosts), and an xtrans bug that made every display collide on the same vsock port is fixed. Connections are treated as local clients (cookie-free), and only the bound VM can reach the listener.

**feature/vsock-wsl2:**  [Implementation Notes 1](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-WSL2-vsock-Fix)

### Raw Input mouse

VcXsrv's Windows input layer (`hw/xwin`) only ever queued absolute pointer events, so XInput2 `XI_RawMotion` was never generated — SDL2 and other XInput2 clients requesting relative mouse mode received no motion data and appeared stuck. (The `SDL_MOUSE_RELATIVE_MODE_WARP=1` workaround functions, but its per-move X11 round trips cause severe audio stutter under WSL.) The patch registers for Windows Raw Input (`WM_INPUT`) at window creation and injects hardware-level relative displacement into the X server's relative-event pipeline, flags absolute events so they no longer pollute raw motion, and sets the master pointer's valuator mode to `Relative` so clients interpret raw values correctly.

### Cursor confinement & hiding

VcXsrv's DDX layer never implemented Windows-side cursor management for X11 pointer grabs: `XGrabPointer` with `confineTo` had no visible effect — the cursor stayed visible and could leave the window freely. The patch hooks the master and slave pointer grab callbacks so that grabs requesting confinement get a properly clipped (`ClipCursor`, with a 1px border guard and a 10ms re-apply timer) and hidden cursor, with Alt+Tab-aware temporary release and restore; implicit grabs and seamless mode are untouched. Same PR and same implementation notes as the Raw Input mouse patch above.

### Empty-mask cursor hiding

X clients hide the cursor by defining a 1x1 all-zero ("empty") cursor — SDL uses this outside of grabs, e.g. dosbox-staging seamless mode. VcXsrv never honored it: the mi cursor layer converts such cursors to NullCursor, `winSetCursor(NULL)` does not hide the Windows cursor in the default mode, and the `emptyMask` rendering path would draw black pixels. The patch lets empty-mask cursors through (`showTransparent`, the xf86 `HARDWARE_CURSOR_SHOW_TRANSPARENT` approach) and renders them fully transparent. Same PR #78.

**feature/raw-input-mouse:**  [Implementation Notes 2](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-SDL2-Relative-Mouse-Mode-Fix),  [Implementation Notes 3](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-Empty-Cursor-Hide-Fix)

---

## Quick Start

Download from the [Releases page](../../releases).

**WSL2 (recommended: vsock transport):**

```bash
# Windows side
vcxsrv.exe :0 -multiwindow -clipboard -wgl -wslvsock

# WSL2 side — one socat forwarder (needs Store WSL 2.0+ and socat ≥ 1.7.4)
socat UNIX-LISTEN:/tmp/.X11-unix/X0,fork,mode=777,forever,retry=10,interval=2 VSOCK-CONNECT:2:106000 &
export DISPLAY=:0
```

XLaunch users can equivalently set `ExtraParams="-wslvsock"` in `config.xlaunch` (the wizard's "Additional parameters for VcXsrv" field). 

[**Full User Guide** (autostart, WSLg coexistence, troubleshooting)](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-WSL2-vsock-User-Guide)

> **WSL1, or any non-WSL setup:** run as usual — the mouse patches need no configuration:
> ```bash
> vcxsrv.exe -multiwindow -clipboard -wgl
> ```

> **Tip:** When running dosbox-staging on WSL, if the cursor stays still in seamless mode (does not follow the system cursor), set `export XDG_CURRENT_DESKTOP=WSL` before launching. SDL2 uses this to decide whether to enable X11 mouse integration.

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

The most useful thing you can do — more than starring this fork — is to test the upstream submissions and leave your results there:

- [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) (mouse patches) and [Issue #77](https://github.com/marchaesen/vcxsrv/issues/77)
- [Issue #80](https://github.com/marchaesen/vcxsrv/issues/80) (vsock transport)

Confirmed test reports from real users are what moves long-running PRs forward, and once merged, everyone gets these fixes in the official builds. Bugs specific to these unofficial builds can be reported in this fork's issue tracker.

---

## Roadmap

- [x] Raw Input mouse (`WM_INPUT` → `XI_RawMotion`)
- [x] Cursor confinement & hiding on grab
- [x] Empty-mask cursor hiding (SDL seamless mode)
- [x] AF_VSOCK transport — zero-config, self-healing WSL2 VM↔host channel
- [ ] Upstream merge of PR #78; vsock PR following Issue #80
- [ ] Clipboard improvements — smoother bidirectional text/image sharing between Windows host and X11 clients

---

## How-to Build

- **English:** [Building VcXsrv on WSL](https://github.com/vivimillin/vcxsrv/wiki/Building-VcXsrv-on-WSL)

---

## Branches

| Branch | Purpose | Default? |
| --- | --- | --- |
| `master` | Clean mirror of upstream | No  |
| `fork-pages` | GitHub homepage, this README | **Yes** |
| `feature/raw-input-mouse` | PR #78 code | No  |
| `feature/vsock-wsl2` | vsock transport code (Issue #80) | No  |

`master` tracks upstream via `reset --hard` and carries no local commits. All fork-specific content lives on `fork-pages`.

---

## License

MIT-style X11 license, same as upstream VcXsrv.

---
---

## 中文简介

**中文** · [English](#readme)

[VcXsrv](https://github.com/marchaesen/vcxsrv)（Windows 平台开源 X server）的非官方构建版本，包含两组补丁，补齐了 VcXsrv 成为 WSL 的一流 Linux GUI 显示方案所差的两块拼图：WSL2 vsock传输 & 相对鼠标模式与光标锁定。

- **WSL2 Hyper-V vsock 传输**——VM 与宿主机的直通通道，取代 TCP / localhost 转发路径：鼠标密集负载下延迟更低、不受 VPN 切换或睡眠唤醒影响、几乎零配置。**让 VcXsrv 在 WSL2 下的显示响应达到 WSLg 和商业软件 X410 的水平，同时保持完全开源。**

- **相对鼠标模式与光标锁定**——硬件级 `XI_RawMotion` 加上正确的 `XGrabPointer` 锁定与消隐，**让 SDL/SDL2 游戏、模拟器、3D/CAD 工具正确捕获鼠标**。鼠标相对模式失灵是 WSL 长期存在的痛点——WSLg（[microsoft/wslg#240](https://github.com/microsoft/wslg/issues/240)、[#521](https://github.com/microsoft/wslg/issues/521)）和 X410 上都存在。

所有补丁均以规范 PR 形式提交上游（[PR #78](https://github.com/marchaesen/vcxsrv/pull/78)；[Issue #80](https://github.com/marchaesen/vcxsrv/issues/80)，PR 随后提交）。上游大约每年发布一次，本 fork 在审核期间提供构建版本——补丁一旦被上游合入即行退役。

[Releases](../../releases) · [上游 PR #78](https://github.com/marchaesen/vcxsrv/pull/78) · [Issue #77](https://github.com/marchaesen/vcxsrv/issues/77) · [Issue #80](https://github.com/marchaesen/vcxsrv/issues/80)

---

## 适用场景

**所有在 WSL2 上运行 Linux GUI 应用的用户——仅 vsock 传输一项就值得试一试。**

上游 VcXsrv 经 TCP（NAT + localhost 转发）与 WSL2 通信，这条路径正是一系列已知问题的根源：鼠标密集负载下的卡顿（拖拽、滚动、SDL2 消息风暴）、睡眠唤醒或 Wi-Fi/VPN 切换后的断连或挂起。

新的 `-wslvsock` 实现了把 X11 流量迁移到 Hyper-V socket——一条专为 VM↔宿主机通信设计、完全绕过 TCP/IP 协议栈的通道：

- **性能**——数据路径上不再有 NAT、localhost 代理和网络协议栈开销；输入密集负载不再卡顿
- **稳定性**——不受 VPN 切换、Wi-Fi 漫游、网卡节电、睡眠唤醒的影响；
- **无需特权**——启动时自动检测运行中的 WSL2 VM，每次 WSL 重启后自动重新检测；不折腾 `DISPLAY` 和宿主机 IP，不需要管理员权限，也不需要 "Hyper-V Administrators" 组成员——而 [X410 可靠检测 WSL2 的方案恰恰需要加入该组](https://x410.dev/cookbook/wsl/using-x410-with-wsl2/)

**所有运行鼠标捕获类 X11 应用的用户**（WSL1/WSL2，或任何以 VcXsrv 为显示服务器的场景）可获得 Windows DDX 层（`hw/xwin`）的两项 X11 协议级修复：

- 请求相对鼠标模式的 **SDL2 应用**（dosbox-staging、AssaultCube、crispy-doom、Quake 系列引擎及其他众多游戏）
- 捕获鼠标用于视角导航的 **3D/CAD 工具**（Blender、FreeCAD）
- 在 X11 上运行时锁定本地光标的 **远程桌面/VNC 查看器**
- 使用鼠标捕获实现宿主机集成的 **模拟器**
- 任何调用 `XGrabPointer` 并传入 confine window 的 X11 客户端

## 横向对比

| | WSLg | X410 | 官方 VcXsrv | **本 fork** |
| --- | --- | --- | --- | --- |
| 开源 | Linux 侧组件开源 | ✗（闭源、付费） | ✓ | ✓ |
| WSL2 传输 | 本地 socket（快） | Hyper-V vsock（快） | TCP 经 NAT / localhost 转发 | **Hyper-V vsock（快）** |
| WSL2 配置 | 无需配置 | 少量配置；可靠检测 WSL2 的 vsock 需加入 "Hyper-V Administrators" 组² | 手动 `DISPLAY` / 宿主机 IP | 一个参数 + 一条 socat 命令，无需管理员 |
| 相对鼠标模式（`XI_RawMotion`） | 众多游戏失灵¹ | 有用户反馈问题；无公开 tracker | 从不生成 | ✓ 正常 |
| 光标锁定与消隐 | 众多游戏失灵¹ | 有用户反馈问题；无公开 tracker | 未实现 | ✓ 正常 |

¹ 公开 issue：[microsoft/wslg#240](https://github.com/microsoft/wslg/issues/240)（游戏中鼠标锁定失效、输入混乱）、[microsoft/wslg#521](https://github.com/microsoft/wslg/issues/521)（游戏无法捕获光标）。

² 据 [X410 官方文档](https://x410.dev/cookbook/wsl/using-x410-with-wsl2/)，其"更可靠检测 WSL2"的 vsock 代码"需要访问 Windows Hyper-V 相关 API 的额外用户权限"（即加入 "Hyper-V Administrators" 组）。本 fork 改用 `wsl.exe -- wslinfo --vm-id` 检测 VM，完全无需提权。

本 fork 并非要替代 WSLg 或 X410 已经做好的部分——重点是**完全开源**的方案不必再以传输更慢、鼠标捕获失灵为代价。

---

## 提供的补丁

| 补丁 | 修复内容 | 上游状态 |
| --- | --- | --- |
| **WSL2 vsock 传输** | hyperv 监听永远匹配不上 WSL2 的 VM；绑定时忽略 display 号；默认即监听 | [Issue #80](https://github.com/marchaesen/vcxsrv/issues/80) — Open，PR 随后 |
| **Raw Input 鼠标** | XInput2 `XI_RawMotion` 消息不生成；SDL2 鼠标相对模式无法工作 | [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) — Open |
| **光标锁定与消隐** | `XGrabPointer` 设置 `confineTo` 无效；光标未锁定且未隐藏 | 同属 [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) — Open |
| **空光标隐藏** | 客户端用全零掩码光标隐藏指针时，屏幕上残留旧光标图像 | 同属 [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) — Open |

非官方构建见 [Releases 页面](../../releases)，每个 release 注明了实际包含的补丁。

### WSL2 vsock 传输（`-wslvsock`）

VcXsrv 多年来一直内置 Hyper-V vsock 传输，但对 WSL2 从未开箱可用：监听绑定的 wildcard VM id 被 WSL2 的 utility VM 拒绝匹配，而所需的确切 VM id 每次 WSL 重启都会变化，且以往只有提权工具才能查到。<br>`-wslvsock` 补上了这个缺口：经 `wsl.exe -- wslinfo --vm-id` 自动检测运行中的 WSL2 VM——**无需管理员权限、无需 "Hyper-V Administrators" 组**（不同于 [X410 采用的 HCS API 方案](https://x410.dev/cookbook/wsl/using-x410-with-wsl2/)）——将监听绑定到该 VM 的 `106000 + display` 端口，并监视 VM 实例，使每次 WSL 重启后约 1–2 秒内自动重绑到新 id，无需重启 VcXsrv。<br>WSL1 或无 WSL2 VM 时静默回退到 TCP，完全兼容原有设置；vsock 默认关闭（无 wildcard 监听暴露、非 Hyper-V 主机无日志噪音）；同时修复了 xtrans 中所有 display 使用同一 vsock 端口的 bug。

**feature/vsock-wsl2:**  [修复笔记1](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-WSL2-vsock-修复)

### Raw Input 鼠标

VcXsrv 的 Windows 输入层（`hw/xwin`）只队列化绝对坐标事件，从不生成 XInput2 `XI_RawMotion`——请求相对鼠标模式的 SDL2 及其他 XInput2 客户端收不到运动数据，表现为卡死（虽然可用变通方案 `SDL_MOUSE_RELATIVE_MODE_WARP=1`，但每次移动一次的 X11 往返消息风暴在 WSL 下会导致画面和音频卡顿）。<br>补丁在窗口创建时注册 Windows Raw Input（`WM_INPUT`），把硬件级相对位移注入 X server 的相对事件管道；给绝对坐标事件加标志防止污染 raw motion；并将 master pointer 的 valuator 模式设为 `Relative`，使客户端正确解释 raw 值。

### 光标锁定与消隐

VcXsrv 的 DDX 层从未为 X11 pointer grab 实现 Windows 端的光标管理：`XGrabPointer` 设置 `confineTo` 后没有任何可见效果——光标未锁定、未隐藏。<br>补丁 hook master/slave pointer 的 grab 回调，使请求锁定的 grab 获得正确的光标裁剪（`ClipCursor`，1px 边框保护 + 10ms 重应用定时器）与消隐，并支持 Alt+Tab 临时释放与恢复；implicit grab 和 seamless 模式不受影响。与上面的 Raw Input 鼠标属同一个 PR。

### 空光标隐藏

X 客户端通过定义 1×1 全零（"空"）光标来隐藏指针——SDL 在非抓取场景使用此法，如 dosbox-staging seamless 模式。VcXsrv 从未正确实现：mi 层把空光标替换为 NullCursor，`winSetCursor(NULL)` 在默认模式下并不隐藏 Windows 光标，emptyMask 渲染路径还会画出黑色像素。补丁放行空掩码光标（`showTransparent`，与 xf86 `HARDWARE_CURSOR_SHOW_TRANSPARENT` 同一做法）并渲染为全透明。同属 PR #78。

**feature/raw-input-mouse:**  [修复笔记2](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-SDL2-Relative-Mouse-Mode-修复),  [修复笔记3](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-Empty-Cursor-Hide-修复)

---

## 快速使用

从 [Releases 页面](../../releases) 下载安装。

**WSL2（推荐 vsock 传输）：**

```bash
# Windows 侧
vcxsrv.exe :0 -multiwindow -clipboard -wgl -wslvsock

# WSL2 侧 —— 一条 socat 转发（需 Store 版 WSL 2.0+、socat ≥ 1.7.4）
socat UNIX-LISTEN:/tmp/.X11-unix/X0,fork,mode=777,forever,retry=10,interval=2 VSOCK-CONNECT:2:106000 &
export DISPLAY=:0
```

XLaunch 用户可等价地在 `config.xlaunch` 里设置 `ExtraParams="-wslvsock"`（即向导的 "Additional parameters for VcXsrv" 输入框）。

[**完整使用指南**（自启动、与 WSLg 共存、故障排查）](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-WSL2-vsock-使用指南)

> **WSL1 或其他非 WSL 场景：** 照常运行即可，鼠标补丁无需任何配置：
> ```bash
> vcxsrv.exe -multiwindow -clipboard -wgl
> ```

> **提示：** 在 WSL 上运行 dosbox-staging 时，如果 seamless 模式下光标静止不动（不跟随系统光标移动），启动前设置 `export XDG_CURRENT_DESKTOP=WSL`。SDL2 依赖此变量决定是否启用 X11 鼠标集成。

---

## 测试情况

鼠标补丁：

| 应用 | 平台 | 模式 | 结果 |
| --- | --- | --- | --- |
| dosbox-staging 0.82 | WSL1/WSL2 | seamless + capture | 通过 |
| AssaultCube 1.3 | WSL1/WSL2 | capture + release | 通过 |
| crispy-doom 5.11 | WSL1/WSL2 | capture + release | 通过 |

vsock 传输（Windows 11 + Store 版 WSL2，Ubuntu）：

| 验证项 | 结果 |
| --- | --- |
| vsock 上的 X11 握手 | `SUCCESS`，免 cookie（FamilyLocal） |
| 双实例 `:11` / `:12` 均带 `-wslvsock` | 106011 / 106012 均可连通（display 偏移修复） |
| 服务端运行期间 WSL VM 真实重启 | 监听自动重绑到新 VM id；客户端重连成功 |
| 默认启动（无 `-wslvsock`） | 无 vsock 监听；TCP 路径无变化（无回归） |

---

## 帮助补丁合入上游

比 star 本 fork 更有用的是：测试上游的提交并留下你的结果：

- [PR #78](https://github.com/marchaesen/vcxsrv/pull/78)（鼠标补丁）与 [Issue #77](https://github.com/marchaesen/vcxsrv/issues/77)
- [Issue #80](https://github.com/marchaesen/vcxsrv/issues/80)（vsock 传输）

真实用户的测试报告是帮助 review 的最有效方式；合入后官方构建就能为更多人使用。针对这些非官方构建本身的 bug，请报到本 fork 的 issue tracker。

---

## 路线图

- [x] Raw Input 鼠标（`WM_INPUT` → `XI_RawMotion`）
- [x] Grab 时光标锁定与消隐
- [x] 空光标隐藏（SDL seamless 模式）
- [x] AF_VSOCK 传输——零配置、自愈的 WSL2 VM↔宿主通道
- [ ] 上游合入 PR #78；vsock PR 随 Issue #80 提交
- [ ] 剪贴板改进——Windows 宿主机与 X11 客户端之间更流畅的双向文本/图像共享

---

## 编译指南

- **中文：** [基于 WSL 的 VcXsrv 编译指南](https://github.com/vivimillin/vcxsrv/wiki/基于-WSL-的-VcXsrv-编译指南)

---

## 分支说明

| 分支 | 用途 | 默认分支？ |
| --- | --- | --- |
| `master` | 上游的干净镜像 | 否 |
| `fork-pages` | GitHub 主页、本 README | **是** |
| `feature/raw-input-mouse` | PR #78 代码 | 否 |
| `feature/vsock-wsl2` | vsock 传输代码（Issue #80） | 否 |

`master` 通过 `reset --hard` 跟踪上游，不携带本地提交。所有 fork 专属内容在 `fork-pages` 上。

---

## 许可证

MIT-style X11 license，与上游 VcXsrv 相同。
