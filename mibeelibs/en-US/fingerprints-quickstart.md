# Quick start: load rules and classify

`mibee-fingerprints-go` is the reference Go engine for the MiBee
fingerprint library: load YAML rules, evaluate collected `Evidence`, and
emit `ServiceIdentity` assertions. This page is the minimal compilable
example.

## Install

```bash
go get github.com/Mi-Bee-Studio/mibee-fingerprints-go@v0.1.0
```

## Minimal example

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

Notes:

- `LoadEmbeddedDefaults` uses the rule set embedded into the binary —
  zero configuration, zero external files.
- `LoadFromDir` loads a custom rule directory; the rule format is defined
  by the [adapter spec](https://github.com/Mi-Bee-Studio/MiBeeSteward/blob/main/docs/fingerprint-spec.md).
- The data shapes (`Evidence` / `ServiceIdentity`) are deliberately
  language-agnostic: any runtime producing/consuming the same JSON
  structures reaches identical classifications.

## Next steps

- [Engine overview](fingerprints-overview.md): design motivation, rule
  model, and the adapter spec.
