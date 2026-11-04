# CLI 参考

`reach` 是单二进制多合一 CLI：被控端与主控端能力都在里面。控制台内部的
各类操作最终也落在这些命令上。

## 命令总览

```
reach desktop  host | connect | monitors | lan-key     远程桌面
reach terminal host | connect                          远程终端
reach tunnel   host | connect                          端口转发
reach ssh                                              SSH server（Windows/Linux）
reach vdd       ...                                    虚拟显示器管理（仅 Windows）
reach login / devices / enroll                         平台账号体系（跨网版能力）
reach --version                                        版本
```

> `reach login / devices / enroll` 属跨网（P2P）版能力，局域网版以局域网
> 发现与配对为准。

## desktop

```sh
# 被控端（常用参数均有默认值）
reach desktop host --bind 0.0.0.0:7777 --fps 20
    --lan-key <hex64>        LAN 访问码（缺省用持久化 key）
    --lan-open               开放模式（任何局域网机器可连）
    --view-only              只读被控端（禁远端输入）
    --tile-cols / --tile-rows / --capture-scale   编码 tile 与捕获比例
    --audio                  启用音频采集
    --record ...             录屏输出（RTSP/ONVIF，详见 --help）
    --virtual-display ...    虚拟显示器策略（off/fallback/on，仅 Windows）
    --dark-backdrop          背景置黑策略
    --file-dir <path>        接收文件保存目录

# 主控端
reach desktop connect --host <ip>:7777
    --lan-key <hex64>        与被控端一致的访问码
    --zoom fit|100|50%       初始显示比例
    --control-pipe <name>    控制管道（控制台嵌入场景用）
    --embedded               嵌入模式（窗口几何交控制台管理）

reach desktop monitors --host <ip>:7777     列出远端显示器
reach desktop lan-key                       查看/生成持久化访问码
```

## terminal

```sh
reach terminal host    --bind 0.0.0.0:7778 [--lan-key <hex64>]
reach terminal connect --host <ip>:7778 [--lan-key <hex64>]
```

## tunnel

```sh
reach tunnel host    --bind 0.0.0.0:7779 [--lan-key <hex64>]
reach tunnel connect --host <ip>:7779 --local 127.0.0.1:2223 --target 127.0.0.1:22
```

`--target` 为被控端视角的地址；连接桌面 host（7777）时每条流带目标头，
由 host 侧按流转发。

## 环境变量（常用）

| 变量 | 作用 |
|---|---|
| `RUST_LOG` | 日志级别（如 `RUST_LOG=info` / `off`） |
| `MIBEE_ADVERTISE_ADDR` | 公网端点公告（跨网场景，局域网版不需要） |

完整环境变量清单见 `docs/env-vars.md`（源码仓）。
