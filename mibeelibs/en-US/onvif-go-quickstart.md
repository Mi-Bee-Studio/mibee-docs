# Quick start: discover and pull a stream

`onvif-go/v2` is an ONVIF client (with a server side too). This page is
the minimal compilable example: connect to a camera, list profiles, and
pick the main / sub streams with the built-in heuristics.

## Install

```bash
go get github.com/mickeyzzc/onvif-go/v2@v2.0.0-rc6
```

> v2 is still at rc stage; the API may shift slightly — pin the exact
> version in production.

## Minimal example

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

Notes:

- `NewClient` accepts an IP or URL; add auth via options like
  `WithCredentials` as needed ([authentication](onvif-go-authentication.md)).
- Stream URIs come from `client.Media().GetStreamURI`, parameterizable via
  `StreamSetup` for RTSP / HTTP transports ([media](onvif-go-media.md)).
- Event subscriptions are one call:
  `client.Events().SubscribeEvents` ([events](onvif-go-events.md)).

## Next steps

- [Discovery](onvif-go-discovery.md): WS-Discovery probes for cameras on
  the network.
- [Media](onvif-go-media.md): profile-selection heuristics and stream URIs.
- [Architecture](onvif-go-architecture.md) /
  [v2 architecture](onvif-go-v2-architecture.md): layering and migration.
