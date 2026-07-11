# VcXsrv — Builds with Mouse & Integration Patches

VcXsrv builds with patches for relative mouse mode, cursor confinement, and integration improvements. Fork of [marchaesen/vcxsrv](https://github.com/marchaesen/vcxsrv). This fork exists to make the patches available while the corresponding upstream PRs are under review.

[Releases](../../releases) · [Upstream PR #78](https://github.com/marchaesen/vcxsrv/pull/78) · [Issue #77](https://github.com/marchaesen/vcxsrv/issues/77)

---

## Who Needs This

Any X11 application running on VcXsrv that uses **XInput2 relative mouse mode** (`XI_RawMotion`) or **`XGrabPointer` with `confineTo`**:

- **SDL2 applications** requesting relative mouse mode (e.g., dosbox-staging, AssaultCube, crispy-doom, Quake-based engines, many other games)
- **3D/CAD tools** that capture the mouse for viewport navigation (e.g., Blender, FreeCAD)
- **Remote desktop / VNC viewers** running on X11 that lock the local cursor
- **Emulators** using mouse capture for host integration
- Any X11 client calling `XGrabPointer` with a confine window

These are X11-protocol-level improvements in VcXsrv's Windows DDX layer (`hw/xwin`). They benefit any program that relies on these standard X11 features — most commonly encountered when running Linux GUI apps via WSL1/WSL2, but applicable to any setup using VcXsrv as the display server.

---

## What This Provides

| Patch | What It Fixes | Upstream Status |
| --- | --- | --- |
| **Raw Input Mouse** | XInput2 `XI_RawMotion` not generated; SDL2 relative mouse mode broken | [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) — Open |
| **Cursor Confinement** | `XGrabPointer` with `confineTo` has no effect; cursor stays visible and free | Same [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) — Open |

> **Note:** Both fixes are part of the same PR #78. They address two parts of one issue — raw input data delivery and the accompanying cursor confinement/hiding behavior.

### Raw Input Mouse

VcXsrv's Windows input layer (`hw/xwin`) only queued `POINTER_ABSOLUTE` events from `WM_MOUSEMOVE`. `XI_RawMotion` events were never generated, so SDL2 and other XInput2 clients requesting relative mouse mode received no motion data and appeared stuck.

The workaround `SDL_MOUSE_RELATIVE_MODE_WARP=1` restores function but causes severe audio stutter under WSL due to X11 round-trip overhead.

This patch:

- Registers for `WM_INPUT` (Raw Input) at window creation to receive hardware-level relative displacement (`lLastX`/`lLastY`)
- Adds `winEnqueueRawMotion()` to inject `POINTER_RELATIVE` events into the DIX layer, producing `XI_RawMotion`
- Adds `POINTER_NORAW` to `winEnqueueMotion()` so absolute coordinates don't pollute raw motion
- Sets the master pointer's valuator mode to `Relative` so XInput2 clients interpret raw values correctly

### Cursor Confinement & Hiding

VcXsrv's DDX layer never implemented Windows-side cursor management for X11 pointer grabs. `XGrabPointer` with `confine_to` had no visible effect — the cursor stayed visible and could leave the window freely.

This patch hooks `ActivateGrab`/`DeactivateGrab` on the master and slave pointer devices to:

- Confine the cursor to the window during `XGrabPointer` via `ClipCursor`
- Hide the system cursor during grab via `ShowCursor(FALSE)`
- Only activate when `grab->confineTo` is set — implicit grabs and seamless mode are unaffected
- Map the X11 grab window to Windows `HWND` via `winGetWindowPriv()` for precise clipping
- 1px `ClipCursor` padding to prevent accidental sizing-border resize
- 10ms timer re-applying `ClipCursor` (defends against Windows silent cancel and cursor skipping on high-DPI mice)
- `WM_ACTIVATEAPP` handling for Alt+Tab aware temporary release and restore

---

## Quick Start

Download from the [Releases page](../../releases) and run as usual:

```bash
vcxsrv.exe -multiwindow -clipboard -wgl
```

> **Tip:** When running dosbox-staging on WSL, if the cursor stays still in seamless mode (does not follow the system cursor), set `export XDG_CURRENT_DESKTOP=WSL` before launching. SDL2 uses this to decide whether to enable X11 mouse integration.

---

## Tested With

| Application | Platform | Mode | Result |
| --- | --- | --- | --- |
| dosbox-staging 0.82 | WSL1/WSL2 | seamless + capture | Pass |
| AssaultCube 1.3 | WSL1/WSL2 | capture + release | Pass |
| crispy-doom 5.11 | WSL1/WSL2 | capture + release | Pass |

---

## Roadmap

- [x] Raw Input mouse (`WM_INPUT` → `XI_RawMotion`)
- [x] Cursor confinement & hiding on grab
- [ ] AF_VSOCK transport — direct VM–host communication channel, eliminating TCP localhost dependency (relevant to WSL2, QEMU, and other VM setups; resolves message queuing stalls and bandwidth limits in WSL2, approaching WSLg-level performance)
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

`master` tracks upstream via `reset --hard` and carries no local commits. All fork-specific content lives on `fork-pages`.

---

## License

MIT-style X11 license, same as upstream VcXsrv.

---
---

## 中文简介

[VcXsrv](https://github.com/marchaesen/vcxsrv) 的构建版本，包含相对鼠标模式、光标锁定和集成改进补丁。Fork 自 [marchaesen/vcxsrv](https://github.com/marchaesen/vcxsrv)。在对应的上游 PR 审核期间，本项目提供预编译版本供需要的用户提前使用。

[Releases](../../releases) · [Upstream PR #78](https://github.com/marchaesen/vcxsrv/pull/78) · [Issue #77](https://github.com/marchaesen/vcxsrv/issues/77)

---

## 适用场景

任何在 VcXsrv 上运行、且使用 **XInput2 相对鼠标模式**（`XI_RawMotion`）或 **`XGrabPointer` 并设置 `confineTo`** 的 X11 应用：

- 请求相对鼠标模式的 **SDL2 应用**（如 dosbox-staging、AssaultCube、crispy-doom、Quake 系列引擎及其他众多游戏）
- 捕获鼠标用于视角导航的 **3D/CAD 工具**（如 Blender、FreeCAD）
- 在 X11 上运行时锁定本地光标的 **远程桌面/VNC 查看器**
- 使用鼠标捕获实现宿主机集成的 **模拟器**
- 任何调用 `XGrabPointer` 并传入 confine window 的 X11 客户端

这些是 VcXsrv Windows DDX 层（`hw/xwin`）的 X11 协议级改进。它们使依赖这些标准 X11 特性的程序正常工作——最常见的使用场景是通过 WSL1/WSL2 运行 Linux GUI 应用，但适用于任何使用 VcXsrv 作为显示服务器的场景。

---

## 提供的补丁

| 补丁  | 修复内容 | 上游状态 |
| --- | --- | --- |
| **Raw Input 鼠标** | XInput2 `XI_RawMotion` 消息不生成；SDL2 鼠标相对模式无法工作 | [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) — Open |
| **光标锁定** | `XGrabPointer` 设置 `confineTo` 无效；光标未锁定且未隐藏 | 同属 [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) — Open |

> 两个修复同属一个 [PR #78](https://github.com/marchaesen/vcxsrv/pull/78)，针对 [Issue #77](https://github.com/marchaesen/vcxsrv/issues/77)，是同一问题的两个方面——原始输入数据传递和配套的光标锁定/消隐行为。

### Raw Input 鼠标

VcXsrv 的 Windows 输入层（`hw/xwin`）只从 `WM_MOUSEMOVE` 队列化 `POINTER_ABSOLUTE` 事件，从不生成 `XI_RawMotion` 消息。请求相对鼠标模式的 SDL2 及其他 XInput2 客户端收不到运动数据，表现为卡死。

变通方案 `SDL_MOUSE_RELATIVE_MODE_WARP=1` 可以恢复功能，但在 WSL 下会导致严重的音频卡顿（X11 往返时间开销导致阻塞）。

此补丁：

- 在窗口创建时注册 `WM_INPUT`（Raw Input），接收硬件级相对位移（`lLastX`/`lLastY`）
- 新增 `winEnqueueRawMotion()` 将 `POINTER_RELATIVE` 事件注入 DIX 层，产生 `XI_RawMotion`
- 给 `winEnqueueMotion()` 添加 `POINTER_NORAW`，防止绝对坐标污染 raw motion
- 将 master pointer 的 valuator 模式设为 `Relative`，使 XInput2 客户端正确解释 raw 值

### 光标锁定与消隐

VcXsrv 的 DDX 层从未为 X11 pointer grab 实现 Windows 端的光标管理。`XGrabPointer` 设置 `confine_to` 后没有任何可见效果——光标未如预期锁定、且未隐藏。

此补丁通过 hook master/slave pointer device 的 `ActivateGrab`/`DeactivateGrab` 来：

- 捕获期间通过 `ClipCursor` 将光标限制在窗口内
- 捕获期间通过 `ShowCursor(FALSE)` 隐藏系统光标
- 仅当 `grab->confineTo` 设置时才激活；implicit grab 和 seamless 模式不受影响
- 通过 `winGetWindowPriv()` 将 X11 grab window 映射到 Windows `HWND` 实现精确裁剪
- 1px `ClipCursor` 内边距防止误触 sizing border 调整窗口大小
- 10ms 定时器重新应用 `ClipCursor`，防御 Windows 静默取消和高 DPI 光标跳过
- 通过 `WM_ACTIVATEAPP` 在 Alt+Tab 时临时释放，激活时恢复

---

## 快速使用

从 [Releases 页面](../../releases) 下载安装，然后照常运行：

```bash
vcxsrv.exe -multiwindow -clipboard
```

> **提示：** 在 WSL 上运行 dosbox-staging 时，如果 seamless 模式下光标静止不动（不跟随系统光标移动），启动前设置 `export XDG_CURRENT_DESKTOP=WSL`。SDL2 依赖此变量决定是否启用 X11 鼠标集成。

---

## 测试情况

| 应用  | 平台  | 模式  | 结果  |
| --- | --- | --- | --- |
| dosbox-staging 0.82 | WSL2 | seamless + capture | 通过  |
| AssaultCube 1.3 | WSL2 | capture + release | 通过  |
| crispy-doom 5.11 | WSL1/WSL2 | capture + release | 通过 |

---

## 路线图

- [x] Raw Input 鼠标（`WM_INPUT` → `XI_RawMotion`）
- [x] Grab 时光标锁定与消隐
- [ ] AF_VSOCK 传输 — 直接的 VM–宿主机通信通道，消除 TCP localhost 依赖（适用于 WSL2、QEMU 及其他虚拟机方案；彻底解决 WSL2 中消息阻塞和带宽问题，获得接近 WSLg 的性能）
- [ ] 剪贴板改进 — Windows 宿主机与 X11 客户端之间更流畅的双向文本/图像共享

---

## 编译指南

- **中文：** [基于 WSL 的 VcXsrv 编译指南](https://github.com/vivimillin/vcxsrv/wiki/基于-WSL-的-VcXsrv-编译指南)

---

## 分支说明

| 分支  | 用途  | 默认分支？ |
| --- | --- | --- |
| `master` | 上游的干净镜像 | 否   |
| `fork-pages` | GitHub 主页、本 README | **是** |
| `feature/raw-input-mouse` | PR #78 代码 | 否   |

`master` 通过 `reset --hard` 跟踪上游，不携带本地提交。所有 fork 专属内容在 `fork-pages` 上。

---

## 许可证

MIT-style X11 license，与上游 VcXsrv 相同。