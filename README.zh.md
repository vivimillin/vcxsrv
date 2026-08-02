# VcXsrv —— 支持 WSL2 vsock 与鼠标相对模式（Unofficial Builds）

[![Release](https://img.shields.io/github/v/release/vivimillin/vcxsrv)](../../releases)
[![License](https://img.shields.io/badge/license-X11-blue)](COPYING)

**中文** · [English Version](README.md)

[VcXsrv](https://github.com/marchaesen/vcxsrv)（Windows 平台开源 X server）的非官方构建版本，包含两组补丁，补齐了 VcXsrv 成为 WSL 的一流 Linux GUI 显示方案所差的两块拼图：WSL2 vsock传输 & 鼠标相对模式与光标锁定。

- **WSL2 Hyper-V vsock 传输**——VM 与宿主机的直通通道，取代 TCP / localhost 转发路径：鼠标密集负载下延迟更低、不受 VPN 切换或睡眠唤醒影响、几乎零配置。**让 VcXsrv 在 WSL2 下 2D 桌面负载的显示响应达到 WSLg 和商业软件 X410 的水平（GPU 加速的 3D/视频仍是 WSLg 的强项），同时保持完全开源。**

- **鼠标相对模式与光标锁定**——硬件级 `XI_RawMotion` 加上正确的 `XGrabPointer` 锁定与消隐，**让 SDL/SDL2 游戏、模拟器、3D/CAD 工具正确捕获鼠标**。鼠标相对模式失灵是 WSL 长期存在的痛点——WSLg（[microsoft/wslg#240](https://github.com/microsoft/wslg/issues/240)、[#521](https://github.com/microsoft/wslg/issues/521)）和 X410 上都存在。

所有补丁均以规范 PR 形式提交上游 [marchaesen/vcxsrv](https://github.com/marchaesen/vcxsrv)（PR #78；Issue #80，PR 随后提交）。上游大约每年发布一次，本项目在审核期间提供构建版本——补丁一旦被上游合入即行退役。

[Releases](../../releases) · [上游 PR #78](https://github.com/marchaesen/vcxsrv/pull/78) · [Issue #77](https://github.com/marchaesen/vcxsrv/issues/77) · [Issue #80](https://github.com/marchaesen/vcxsrv/issues/80)

---

## 适用场景

**所有在 WSL2 上运行 Linux GUI 应用的用户——仅 vsock 带来的性能提升就值得试一试。** 上游 VcXsrv 经 TCP（NAT + localhost 转发）与 WSL2 通信，这条路径正是一系列已知问题的根源：鼠标密集负载下的图像声音卡顿（拖拽、滚动、SDL2 消息风暴）、睡眠唤醒或 Wi-Fi/VPN 切换后的断连或挂起。`-wslvsock` 把 X11 流量迁移到 Hyper-V socket——一条专为 VM↔宿主机通信设计、完全绕过 TCP/IP 协议栈的通道：

- **性能**——数据路径上不再有 NAT、localhost 代理和网络协议栈开销；输入密集负载不再卡顿
- **稳定性**——不受 VPN 切换、Wi-Fi 漫游、网卡节电、睡眠唤醒的影响
- **零配置、零特权**——启动时自动检测运行中的 WSL2 VM，每次 WSL 重启后自动重新检测；不折腾 `DISPLAY` 和宿主机 IP，不需要管理员权限，也不需要 "Hyper-V Administrators" 组成员——而 [X410 可靠检测 WSL2 的方案恰恰需要加入该组](https://x410.dev/cookbook/wsl/using-x410-with-wsl2/)

**所有运行鼠标捕获类 X11 应用的用户**（WSL1/WSL2，或任何以 VcXsrv 为显示服务器的场景）可获得 Windows DDX 层（`hw/xwin`）的两项 X11 协议级修复：

- 请求鼠标相对模式的 **SDL2 游戏**（dosbox-staging、AssaultCube、crispy-doom、Quake 系列引擎等）
- 捕获鼠标用于视角导航的 **3D/CAD 工具**（Blender、FreeCAD）
- 锁定本地光标的 **远程桌面/VNC 查看器与模拟器**——任何调用 `XGrabPointer` 并传入 confine window 的 X11 客户端

## 横向对比

| | WSLg | X410 | 官方 VcXsrv | **本项目** |
| --- | --- | --- | --- | --- |
| 开源 | Linux 侧组件开源 | ✗（闭源、付费） | ✓ | ✓ |
| WSL2 传输 | 本地 socket（快） | Hyper-V vsock（快） | TCP 经 NAT / localhost 转发 | **Hyper-V vsock（快）** |
| WSL2 配置 | 无需配置 | 少量配置；可靠检测 WSL2 的 vsock 需加入 "Hyper-V Administrators" 组² | 手动 `DISPLAY` / 宿主机 IP | 一个参数 + 一条 socat 命令，无需管理员 |
| 鼠标相对模式（`XI_RawMotion`） | 众多游戏失灵¹ | 有用户反馈问题；无公开 tracker | 从不生成 | ✓ 正常 |
| 光标锁定与消隐 | 众多游戏失灵¹ | 有用户反馈问题；无公开 tracker | 未实现 | ✓ 正常 |

> ¹ 公开 issue：[microsoft/wslg#240](https://github.com/microsoft/wslg/issues/240)（游戏中鼠标锁定失效、输入混乱）、[microsoft/wslg#521](https://github.com/microsoft/wslg/issues/521)（游戏无法捕获光标）。<br>
> ² 据 [X410 官方文档](https://x410.dev/cookbook/wsl/using-x410-with-wsl2/)，其"更可靠检测 WSL2"的 vsock 代码"需要访问 Windows Hyper-V 相关 API 的额外用户权限"（即加入 "Hyper-V Administrators" 组）。本项目改用 `wsl.exe -- wslinfo --vm-id` 检测 VM，完全无需提权。

本项目并非要替代 WSLg 或 X410 已经做好的部分——重点是**完全开源**的方案不必再以传输更慢、鼠标捕获失灵为代价。

---

## 提供的补丁

| 补丁  | 修复内容 | 上游状态 | 实现笔记 |
| --- | --- | --- | --- |
| **Raw Input 鼠标** | XInput2 `XI_RawMotion` 消息不生成；SDL2 鼠标相对模式无法工作 | [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) — Open | [修复笔记.1](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-SDL2-Relative-Mouse-Mode-修复) |
| **光标锁定与消隐** | `XGrabPointer` 设置 `confineTo` 无效；光标未锁定且未隐藏 | 同属 [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) — Open | [修复笔记.1](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-SDL2-Relative-Mouse-Mode-修复) |
| **空光标隐藏** | 客户端用全零掩码光标隐藏光标时，屏幕上残留旧光标图像 | 同属 [PR #78](https://github.com/marchaesen/vcxsrv/pull/78) — Open | [修复笔记.2](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-Empty-Cursor-Hide-修复) |
| **WSL2 vsock 传输** | 实现零配置监听 WSL2；修正原 hyperv 监听匹配不上 WSL2 的 VM；修正 display 号被忽略 | [Issue #80](https://github.com/marchaesen/vcxsrv/issues/80) — Open，PR 随后 | [修复笔记.3](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-WSL2-vsock-修复) |

> 非官方构建见 [Releases 页面](../../releases)，每个 release 注明了实际包含的补丁。各补丁的详细设计与实现分析见上方"**实现笔记**"列的 wiki 链接。

---

## 快速使用

1. **安装** [Releases 页面](../../releases)的最新构建。

2. **启动服务端（Windows 侧）：**
   ```bash
   # WSL2 —— 推荐： vsock 传输 & 鼠标修正
   vcxsrv.exe :0 -multiwindow -clipboard -wgl -wslvsock

   # WSL1 或其他非 WSL 场景 —— 仅鼠标修正，无需任何额外配置
   vcxsrv.exe -multiwindow -clipboard -wgl
   ```
   > XLaunch 用户：等价地在 `config.xlaunch` 中设置 `ExtraParams="-wslvsock"`
   > （即向导的 "Additional parameters for VcXsrv" 输入框）。

3. **连接（WSL2 侧）** —— 一条 socat 转发（需 Store 版 WSL 2.0+、socat ≥ 1.7.4）：
   ```bash
   socat UNIX-LISTEN:/tmp/.X11-unix/X0,fork,mode=777,forever,retry=10,interval=2 VSOCK-CONNECT:2:106000 &
   export DISPLAY=:0
   ```
   > 把这两行加入 `~/.bashrc`，每个新终端即可开箱即用。

4. **完成** 登录自启动、与 WSLg 共存、故障排查等，参见 [https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-WSL2-vsock-User-Guide](https://github.com/vivimillin/vcxsrv/wiki/VcXsrv-WSL2-vsock-User-Guide)。
   > **提示：** 在 WSL 上运行 DOSBox-Staging 时，如果 seamless 模式下光标静止不动（不跟随系统光标移动），启动前设置 `export XDG_CURRENT_DESKTOP=WSL`。SDL2 依赖此变量决定是否启用 X11 鼠标集成。

---

## 测试情况

鼠标修正：

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

比 star 本仓库更有用的是：测试上游的提交并留下你的结果：

- [PR #78](https://github.com/marchaesen/vcxsrv/pull/78)（鼠标修正）与 [Issue #77](https://github.com/marchaesen/vcxsrv/issues/77)
- [Issue #80](https://github.com/marchaesen/vcxsrv/issues/80)（vsock 传输）

真实用户的测试报告是帮助 review 的最有效方式；合入后官方构建就能为更多人使用。针对这些非官方构建本身的 bug，请报到本仓库的 issue tracker。

> 开发、issue 与发布均在本仓库进行；上游 PR 经影子仓库 [vivimillin/vcxsrv-upstream](https://github.com/vivimillin/vcxsrv-upstream) 提交。

---

## 路线图

**已完成**
- [x] Raw Input 鼠标（`WM_INPUT` → `XI_RawMotion`）
- [x] Grab 时光标锁定与消隐
- [x] 空光标隐藏（SDL seamless 模式）
- [x] AF_VSOCK 传输——零配置、自愈的 WSL2 VM↔宿主通道

**待完成 / 规划**
- [ ] 上游合入 PR #78；vsock PR 随 Issue #80 提交
- [ ] 剪贴板改进——Windows 宿主机与 X11 客户端之间更流畅的双向文本/图像共享

---

## 从源码构建

- **克隆包含所需补丁的分支**

   ```bash
   # WSL2 vsock 传输 & 鼠标修正 (Issue #80 & PR #78)
   git clone -b feature/vsock-wsl2 https://github.com/vivimillin/vcxsrv.git

   # 仅鼠标修正 (PR #78)
   git clone -b feature/raw-input-mouse https://github.com/vivimillin/vcxsrv.git
   ```

- **完整步骤指南：** [基于 WSL 的 VcXsrv 编译指南](https://github.com/vivimillin/vcxsrv/wiki/基于-WSL-的-VcXsrv-编译指南)

---

## 分支说明

| 分支 | 用途 | 默认分支？ |
| --- | --- | --- |
| `master` | 上游的干净镜像 | 否 |
| `pages` | GitHub 主页、本 README + 上游代码 | **是** |
| `feature/raw-input-mouse` | PR #78 代码 | 否 |
| `feature/vsock-wsl2` | vsock 传输代码（Issue #80） | 否 |

`master` 通过 `reset --hard` 跟踪上游，不携带本地提交。所有项目专属内容在 `pages` 上。

---

## 许可证

MIT-style X11 license，与上游 VcXsrv 相同。

