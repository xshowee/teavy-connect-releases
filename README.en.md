# Teavy Connect

[简体中文](README.md)

Teavy Connect is the free Mac companion for Teavy, the remote desktop app for iPhone and iPad. With it, Teavy can use the video stream (HEVC picture and sound) to reach this Mac from outside your local network, and you no longer need to expose Screen Sharing's port 5900 to the internet.

- It connects to Screen Sharing on the Mac itself on behalf of the remote Teavy, then relays everything over one encrypted connection (QUIC, falling back to TCP when UDP is blocked).
- Only devices paired by scanning a QR code can connect. Unpaired connections are closed before a single byte is relayed and never reach Screen Sharing; your macOS password remains a second line of defense.
- The background agent runs as a dedicated unprivileged user, `_teavyconnect`, and keeps listening after a restart, even before anyone logs in.

This repository only hosts Teavy Connect's installers and update feed. It contains no source code.

## Install

1. Requires macOS 15 or later.
2. Download `Teavy-Connect-<version>.dmg` from [Releases](https://github.com/xshowee/teavy-connect-releases/releases/latest), open it and drag Teavy Connect to Applications.
3. Open Teavy Connect; it lives in the menu bar. The first time, it takes you to System Settings › General › Login Items & Extensions; allow Teavy Connect's background item.
4. Turn on Screen Sharing in System Settings › General › Sharing.
5. Choose "Pair New Device…" from the menu and scan the QR code with Teavy (or enter the pairing code in Teavy). Then open this Mac from Teavy's connection list as usual.

Teavy Connect checks for updates once a day, downloads them in the background and installs them when no connection is in use. "Check for Updates…" in the menu checks right away.

**Your Mac can't be reached while it sleeps.** To stay reachable, keep it awake by other means, for example with [Caffy](https://github.com/xshowee/caffy).

## Reaching your Mac from outside

Teavy Connect opens a single port, **5950 by default, the same number for UDP and TCP** (you can change it in Settings). Making that port reachable from the internet is up to you; the [Chinese guide](README.md#从局域网外连) has full examples for a router port forward, frp with Lucky, and direct IPv6 without a forwarder. Whichever you choose:

- **Forward both UDP and TCP.** The video stream uses QUIC over UDP. With TCP alone you can still connect, but only with the framebuffer encoding: less smooth, and no sound.
- **Stop exposing Screen Sharing's port 5900.** Remove any old port forward or frp proxy for 5900; forward only Teavy Connect's port.
- **With frp, forward both protocols at every hop**: a `tcp` and a `udp` proxy with the same `remotePort` in frpc, and both UDP and TCP from your forwarder (Lucky or similar) to frps.
- **If QUIC never connects and Teavy keeps falling back to TCP**: a forwarder whose UDP rule listens on a wildcard address, on a router whose WAN has several IPv6 addresses, may send replies from a different address, and the client's network drops them. Set the rule's listen address to the one your domain resolves to, or keep a single IPv6 address on the WAN. The same applies to frps, which relays UDP itself: when the frps machine has several IPv6 addresses, or runs a TUN proxy such as sing-box or Clash that takes over the default route, frps replies from another address (even the TUN's private one). If frps runs on a Mac that also has Teavy Connect, use the UDP relay below instead of changing frps. Otherwise set `proxyBindAddr` in frps.toml to the address your domain resolves to (it applies to every proxy on that frps), or keep the TUN proxy away from this port's replies.
- **frps on a Mac with Teavy Connect: UDP relay.** For example, a Mac mini at home runs frps and Teavy Connect, and a MacBook on the go connects back with frpc. In the MacBook's frpc, map Teavy Connect's TCP port to remote port 59500 and its UDP port to remote port 15950. On the Mac mini, add a relay in Teavy Connect › Settings › UDP Relay for frp: `59500 → 127.0.0.1:15950`. It listens on every address, replies from the address each packet arrived on, and passes packets to frps's UDP proxy port on that Mac only. Allow both UDP and TCP 59500 to the Mac mini (TCP reaches frps, UDP reaches Teavy Connect), don't open 15950, and enter `home.example.com:59500` as the MacBook's External Address. Both Macs and Teavy on iPhone or iPad need versions that support this; with an older Teavy, the MacBook sends packets too large for the path and QUIC connects without a picture. frp carries each UDP packet inside its frpc-to-frps connection (TCP with TLS by default), so one lost packet holds back the ones after it and the picture may stall briefly; connect directly when you can.
- Enter the domain and port that reach this port from outside (for example `home.example.com:5950`) as the External Address in Teavy Connect's Settings; your iPhone remembers it when pairing.

## Uninstall

Choose "Uninstall Background Agent…" from the menu, then delete Teavy Connect from Applications. Pairings and the agent's identity stay in `/Library/Application Support/Teavy Connect` so a reinstall keeps working; delete that folder with administrator rights if you no longer need it. The hidden `_teavyconnect` user can't be removed (deleting user records requires Full Disk Access); it does no harm and is reused on reinstall.
