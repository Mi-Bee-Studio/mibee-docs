# 快速开始：加载规则并分类

`mibee-fingerprints-go` 是 MiBee 指纹库的参考 Go 引擎：加载 YAML 规则、
对采集到的 `Evidence` 求值、产出 `ServiceIdentity` 断言。本页给最小
可编译示例。

## 安装

```bash
go get github.com/Mi-Bee-Studio/mibee-fingerprints-go@v0.1.0
```

## 最小示例

```go
package main

import (
	"fmt"
	"log"

	fp "github.com/Mi-Bee-Studio/mibee-fingerprints-go"
)

func main() {
	rc := &fp.RuleClassifier{}
	if err := rc.LoadEmbeddedDefaults(); err != nil {
		log.Fatal(err)
	}
	evidence := []fp.Evidence{{
		Source:     "http-banner",
		Kind:       "banner",
		IP:         "192.0.2.10",
		Port:       80,
		Protocol:   "tcp",
		RawData:    map[string]string{"server": "ExampleHTTPd"},
		Confidence: 0.8,
	}}
	for _, id := range rc.Classify(evidence) {
		fmt.Printf("%s :%d/%s confidence=%.2f\n", id.Service, id.Port, id.Protocol, id.Confidence)
	}
}
```

要点：

- `LoadEmbeddedDefaults` 使用编译进二进制的内嵌规则集——零配置、零
  外部文件依赖。
- 也可用 `LoadFromDir` 加载自定义规则目录；规则格式见
  [适配器规范](https://github.com/Mi-Bee-Studio/MiBeeSteward/blob/main/docs/fingerprint-spec.md)。
- 数据形状（`Evidence` / `ServiceIdentity`）刻意语言无关：任何运行时
  按同一 JSON 结构产出 / 消费，分类结果一致。

## 下一步

- [引擎总览](fingerprints-overview.md)：设计动机、规则模型与适配器规范。
