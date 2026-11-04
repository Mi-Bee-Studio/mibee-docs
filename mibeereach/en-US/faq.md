# FAQ & troubleshooting

## Discovery & connection

**Q: The controller's device list doesn't show the controlled machine?**
Check in order: ① same subnet on both ends (inter-VLAN won't work); ② the
host is running (Windows: tray; Linux: `ss -lunp | grep 7777`); ③ the
router has AP/client isolation enabled (blocks broadcast discovery). If all
else fails, use Quick Connect with `ip:port`.

**Q: Connect times out?**
Allow the corresponding **UDP** port through the controlled machine's
firewall (desktop 7777 / terminal 7778 / tunnel 7779). It's UDP, not TCP —
probing 7777 with a TCP tool will always show closed; that's a false
negative.

**Q: Auth rejected?**
The LAN keys differ. Check `reach desktop lan-key` on the controlled side
and align the controller's stored key (console field or `--lan-key`).

**Q: The controlled machine "disappeared" after a network change?**
Beacon discovery covers the same segment only. In a new segment connect via
`ip:port`; beacons broadcast every 3 seconds and the controller re-discovers
when you return to that segment.

## Sessions

**Q: Stutter / low fps?**
Switch the quality preset to data-saver (`Ctrl+Alt+Q`); prefer 5 GHz Wi-Fi;
check for heavy load on the controlled machine. Bitrate adaptation degrades
smoothly on weak links and climbs back quickly.

**Q: The stats overlay shows FPS 0 on a static desktop?**
That's the real value — no encoding frames are produced while the picture
is unchanged (capture parks to save CPU); RTT/rate keep refreshing from the
ICE/control flow.

**Q: Connected but the remote screen "doesn't move"?**
Check whether **dark backdrop** (`Ctrl+Alt+B`) or a host `--record` policy
blanked the physical output — designed to prevent shoulder-surfing and save
bandwidth; it's expected behavior.

**Q: Clipboard / file transfer doesn't work?**
The pairing grant must include clipboard and files; restarting the host
(tray exit) and reconnecting once helps. Linux controlled sides need a live
desktop session (Wayland/X11).

**Q: The macOS console won't open?**
Right-click → **Open** to pass Gatekeeper (regular notice for ad-hoc
signed bundles on first launch).

## Misc

**Q: Autostart on Linux?**
Write the host command as a systemd unit with `Restart=always`; sketch:

```ini
[Service]
ExecStart=/opt/reach/reach desktop host --bind 0.0.0.0:7777 --fps 20
Restart=always
```

**Q: Cross-network (away from home) connections?**
The v0.1.0 public release is the LAN direct edition; cross-network
capability is on the roadmap for later versions.

**Q: Licensing & commercial use?**
Proprietary — see the bundled `LICENSE` and the product page.
