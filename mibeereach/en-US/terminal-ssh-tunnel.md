# Terminal, SSH & port forwarding

Three "non-desktop" remote capabilities share the same transport (LAN QUIC
+ pairing auth) and the same host process model on the controlled side.

## Remote terminal

Starts a remote shell on the controlled machine (ConPTY on Windows, PTY on
Linux); the controller opens it as an embedded xterm.js tab or a standalone
window:

```sh
# Controlled side
reach terminal host --bind 0.0.0.0:7778
# Controller side
reach terminal connect --host 192.168.1.10:7778
```

Tabs support font size (Settings), popping out into a window, and copy &
paste. Session close has FIN-drain semantics — output from a remote process
that exits lands completely before the tab closes, never truncated.

## Remote SSH

The controlled side ships a built-in SSH server (port 2222); any standard
`ssh` client connects directly — keys / agent / passwords all work as
usual, credentials are never stored by MiBeeReach.

The console's **SSH sessions** card turns it into one click: store
"device + username + target" presets (the default target `127.0.0.1:22` is
the machine's own sshd; any host:port reachable *from that machine* also
works), then click to auto-establish the tunnel and open an embedded
terminal tab.

## Port forwarding

Expose an intranet TCP service to a controller-local port through the
tunnel:

```sh
# Controlled side
reach tunnel host --bind 0.0.0.0:7779
# Controller side: local 2223 → 127.0.0.1:22 as seen from the controlled machine
reach tunnel connect --host 192.168.1.10:7779 --local 127.0.0.1:2223 --target 127.0.0.1:22
# then: ssh -p 2223 anyone@127.0.0.1
```

- `--target` is interpreted **from the controlled machine's perspective**
  (`127.0.0.1:80` = its own port 80);
- The desktop host (7777) also carries forwarding semantics: `connect
  --target` against it forwards every stream per its target header — one
  host port, many purposes;
- The console's **Port forward** dialog (the forward button on a device
  card) is equivalent to the commands above.

## Session monitor

Every active session (desktop / terminal / SSH / tunnel) shows up in the
**Session monitor** collapsed card on the Connect page: uptime, up/down
speed, RTT, FPS, bitrate — 3-second refresh, one-click stop.
