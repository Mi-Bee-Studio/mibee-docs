# Pairing & LAN access key

The security model in one sentence: **pairing required** — connections
without credentials are rejected outright; connections that present the
LAN access key work without prompts; session permissions (control /
clipboard / files) are authorized separately, plus a view-only mode.

## LAN access key

The controlled side **generates and persists a 64-hex-digit access key on
first start**; every connection must present it (challenge-response auth
over QUIC with embedded TLS). The controller can carry it in three places:

- The **LAN access key** field in the console's Quick Connect row (remembered
  locally, re-filled automatically);
- Device-card connections from LAN discovery (the console remembers the
  last key that worked);
- The CLI `--lan-key` parameter.

Managing it on the controlled machine:

```sh
reach desktop lan-key                 # show current key (generates if absent)
reach desktop host --bind 0.0.0.0:7777 --lan-key 00112233...ff   # explicit key
reach desktop host --bind 0.0.0.0:7777 --lan-open               # open mode
```

| Mode | Behavior |
|---|---|
| Default (persisted key) | Generated on first start; controllers holding the same key may connect |
| Explicit `--lan-key` | This run uses the given key (overrides the persisted value) |
| `--lan-open` | Open mode: any LAN machine may connect (**trusted networks only**) |

Key rotation: generating/setting a new key on the controlled side invalidates
the old one immediately — update the stored key on every controller.

## Auth gate & fallback

The LAN auth gate rejects at **connection time** (not a network failure):
when the controller sees that error and the same connection has other
reachable paths configured, it falls through them in order (the v0.1.0
LAN edition has LAN paths only). Terminal and tunnel behave identically.

## Session-level permissions

Session permissions are granted per capability: **control** (input
injection), **clipboard**, **files**; plus **view-only mode** disabling all
input. The controller's connect entry and parameters decide the grant;
the controlled side enforces it.

## Transport security

LAN transport is QUIC (with TLS 1.3 embedded); the handshake carries the
access-key challenge-response — no plaintext on the wire. Clipboard /
files / audio / video all ride multiplexed streams over that one encrypted
channel.
