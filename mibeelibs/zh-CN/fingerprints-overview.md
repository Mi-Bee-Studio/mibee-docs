# mibee-fingerprints-go

MiBee 指纹库的参考 Go 引擎：加载 YAML 规则文件，对采集到的
`Evidence` 逐条求值，产出 `ServiceIdentity` 断言。实现与适配器规范
([fingerprint-spec.md](https://github.com/Mi-Bee-Studio/MiBeeSteward/blob/main/docs/fingerprint-spec.md))一致。

## 安装

```bash
go get github.com/Mi-Bee-Studio/mibee-fingerprints-go@v0.1.0
```

## 用法

```go
import fp "github.com/Mi-Bee-Studio/mibee-fingerprints-go"

// 加载内嵌规则（编译进二进制——零配置）。
rc := &fp.RuleClassifier{}
if err := rc.LoadEmbeddedDefaults(); err != nil {
    log.Fatal(err)
}

// 对证据分类。
identities := rc.Classify(evidence)
for _, id := range identities {
    fmt.Printf("%s:%d → %s (conf=%.2f brand=%s)\n",
        ip, id.Port, id.Service, id.Confidence, id.Metadata["inferred_brand"])
}
```

## 类型

```go
type Evidence struct {
    Kind       string            // "banner"、"snmp"、"http"、"tls" 等
    Port       int
    RawData    map[string]string // 协议相关载荷
    Confidence float64           // [0,1]
    // ……（见 fingerprint.go）
}

type ServiceIdentity struct {
    Service    string            // "ssh"、"http"、"camera" 等
    Port       int
    Confidence float64
    Metadata   map[string]string // brand、version、os_type 等
    // ……
}
```

## 规则格式

规则 YAML 文件与规范格式说明见 [MiBee Steward 仓库](https://github.com/Mi-Bee-Studio/MiBeeSteward)的
`docs/fingerprint-spec.md`。

## 测试

```bash
go test ./...     # 全部测试，含 2554 条规则的完整语料加载
go test -race ./...
```

## 许可证

本模块**按层双许可**，与 [MiBee Steward 主仓库](https://github.com/Mi-Bee-Studio/MiBeeSteward)保持一致：

| 层 | 许可证 |
|---|---|
| 源代码（`*.go`） | [GNU AGPLv3](https://www.gnu.org/licenses/agpl-3.0)（或更新版本） |
| 指纹语料（`fingerprint-assets/*.yaml`） | [CC-BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) |

两层均为 copyleft：衍生 Go 代码须以 AGPLv3（或更新版本）发布，衍生指纹语料
须以 CC-BY-SA 4.0 发布。若 AGPLv3 无法覆盖你的源代码使用场景（闭源衍生、
不开源之 SaaS 等），可洽商业许可——见主仓库的
[LICENSE-COMMERCIAL.md](https://github.com/Mi-Bee-Studio/MiBeeSteward/blob/main/LICENSE-COMMERCIAL.md)。

AGPLv3 全文见 [LICENSE](https://github.com/Mi-Bee-Studio/mibee-fingerprints-go/blob/main/LICENSE)，
第三方署名见 [NOTICE](https://github.com/Mi-Bee-Studio/mibee-fingerprints-go/blob/main/NOTICE)。
语料来源（Rapid7 Recog Apache-2.0、IEEE OUI、IANA PEN 及 nmap NPSL 排除项）亦记录于 NOTICE。
