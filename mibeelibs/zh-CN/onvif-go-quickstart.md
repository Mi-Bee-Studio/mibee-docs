# 快速开始：发现并取流

`onvif-go/v2` 是 ONVIF 客户端（也含服务端）。本页给最小可编译示例：
连接一台相机、列出 profile、用启发式选出主 / 子码流。

## 安装

```bash
go get github.com/mickeyzzc/onvif-go/v2@v2.0.0-rc6
```

> v2 仍是 rc 阶段，API 可能微调；生产固定到具体版本号。

## 最小示例

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/mickeyzzc/onvif-go/v2/onvif"
)

func main() {
	client, err := onvif.NewClient("192.0.2.10")
	if err != nil {
		log.Fatal(err)
	}
	profiles, err := client.Media().GetProfiles(context.Background())
	if err != nil {
		log.Fatal(err)
	}
	mainToken := onvif.SelectMainProfile(profiles)
	subToken := onvif.SelectSubProfile(profiles, mainToken)
	fmt.Println("main profile:", mainToken, "sub profile:", subToken)
}
```

要点：

- `NewClient` 接受 IP 或 URL；鉴权用 `WithCredentials` 等选项按需追加
  （[鉴权与安全](onvif-go-authentication.md)）。
- 取流地址用 `client.Media().GetStreamURI`，可经 `StreamSetup` 指定
  RTSP / HTTP 等传输（[媒体与码流](onvif-go-media.md)）。
- 事件订阅一句话搞定：`client.Events().SubscribeEvents`（[事件](onvif-go-events.md)）。

## 下一步

- [设备发现](onvif-go-discovery.md)：WS-Discovery 探测网内相机。
- [媒体与码流](onvif-go-media.md)：profile 选择启发式与流 URI。
- [架构](onvif-go-architecture.md) / [v2 架构](onvif-go-v2-architecture.md)：分层与迁移。
