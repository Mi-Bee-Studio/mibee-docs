# 快速开始

目标：**十分钟内跑通第一次远程桌面**。路径：被控端装 Host → 主控端开控制台 → 自动发现 → 连接。

## 第一步：被控端安装 Host

### Windows

1. 到 [发布页](https://github.com/mickeyzzc/releases/releases/tag/mibeereach-v0.1.0)
   下载 `MiBeeReach-Host_0.1.0_x64-setup.exe`。
2. 运行安装（按用户安装到 `%LOCALAPPDATA%\MiBeeReach-Host`，可选开机自启）。
3. 完成后 host 以托盘常驻，监听 `0.0.0.0:7777`。

### Linux（x86_64 / arm64）

```sh
tar xzf mibeereach_linux_<arch>.tar.gz
./reach desktop host --bind 0.0.0.0:7777 --fps 20
```

需要跨会话常驻时，把该命令包进你惯用的服务管理方式（systemd user unit、
tmux、rc.local 等）。

## 第二步：主控端安装控制台

### Windows

下载 `mibeereach_windows_amd64.zip`，解压到任意目录，运行
`reach-config.exe`。（旁边的 `reach.exe` 是控制台拉起会话用的引擎，
**两个文件保持同目录**。）

### macOS（Apple Silicon）

下载 `mibeereach_macos_aarch64.tar.gz`，把 `MiBeeReach.app`（控制台）与
`MiBeeReach Viewer.app` 拖入「应用程序」。首次打开被 Gatekeeper 拦截时：
右键 → **打开** 放行（ad-hoc 签名包的常规提示）。

## 第三步：连接

1. 打开控制台，默认落在**连接**页。
2. 同网段的被控端会在几秒内**自动出现**在设备区——点「连接」。
3. 没出现就手输：快速连接框填 `ip:port`（如 `192.168.1.10:7777`）→ **连接**。
4. 远程桌面以页签形式在**工作区**打开——键盘鼠标、剪贴板、文件、音频即插即用。

## 默认端口速查

| 服务 | 端口 | 协议 |
|---|---|---|
| 远程桌面 host | 7777 | UDP (QUIC) |
| 远程终端 host | 7778 | UDP (QUIC) |
| 隧道 host | 7779 | UDP (QUIC) |

被控端防火墙放行对应 **UDP** 端口即可；连接不上先查这里。

## 下一步

- [控制台巡礼](console.md)——三个页面各自管什么
- [配对与 LAN 访问码](pairing.md)——把「谁能连」管起来
- [远程桌面会话](desktop-session.md)——快捷键与能力全表
