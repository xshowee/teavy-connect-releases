# Teavy Connect

[English](README.en.md)

Teavy Connect 是 iPhone、iPad 远程桌面应用 Teavy 的 Mac 配套应用，免费。装上之后，iPhone 和 iPad 上的 Teavy 在局域网外也能用视频流（HEVC 画面和声音）连这台 Mac，屏幕共享的 5900 端口也不用再暴露到公网。

- 它在 Mac 本机替远端的 Teavy 连屏幕共享，再经一条加密连接（QUIC，UDP 不通时退回 TCP）转给 iPhone 或 iPad。
- 只有扫码配对过的设备才能连。没配对的连接在转发任何字节之前就被关掉，碰不到屏幕共享；macOS 账户密码仍是第二道防线。
- 后台代理以专用的无特权用户 `_teavyconnect` 运行，Mac 重启后、还没人登录时也在监听。

这个仓库只放 Teavy Connect 的安装包和更新源，不放源码。

## 安装

1. 需要 macOS 15 或更新版本。
2. 从 [Releases](https://github.com/xshowee/teavy-connect-releases/releases/latest) 下载 `Teavy-Connect-<版本>.dmg`，打开后把 Teavy Connect 拖进“应用程序”。
3. 打开 Teavy Connect，它出现在菜单栏。第一次打开会带你到“系统设置 › 通用 › 登录项与扩展”，允许 Teavy Connect 的后台项目。
4. 在“系统设置 › 通用 › 共享”里打开“屏幕共享”。
5. 菜单里选“配对新设备…”，用 Teavy 扫二维码（或在 Teavy 里输入配对码）。之后照常从 Teavy 的连接列表点开这台 Mac。

Teavy Connect 每天检查一次更新，有新版本时在后台下载，没有连接在用时自动装上。菜单里的“检查更新…”可以马上检查。

**Mac 睡眠时无法连接。**要随时能连，请另外安排防休眠，例如用 [Caffy](https://github.com/xshowee/caffy)。

## 从局域网外连

Teavy Connect 只开放一个端口，默认 **5950，UDP 和 TCP 同号**（可以在设置里改）。怎么从公网到达这个端口由你自己安排，下面是三种常见做法。不论哪种：

- **UDP 和 TCP 都要转发。**视频流走 QUIC（UDP）；只转发 TCP 也能连，但只能用帧缓冲编码，画面没有视频流流畅，也没有声音。
- **不要再把屏幕共享的 5900 端口暴露到公网。**以前为屏幕共享做的 5900 端口映射或 frp 代理，有了 Teavy Connect 之后就删掉，从外面只转发 Teavy Connect 的端口。
- 在 Teavy Connect 的设置里把“外部地址”填成从外面到达这个端口的域名加端口（例如 `home.example.com:5950`），配对时 Teavy 会记下它。

下面的域名、地址和端口都是示例，换成你自己的。

### 做法一：路由器端口映射（有公网 IPv4）

在路由器上把公网的 5950 映射到这台 Mac 的局域网地址的 5950，**协议选 TCP 和 UDP 两项**（有的路由器要分两条规则建）。域名用路由器或 DDNS 服务维护。

### 做法二：frp 加 Lucky

适合 Mac 不在家里（比如随身带出门的 MacBook），由 Mac 上的 frpc 连回家里的 frps，再经 Lucky 之类的转发器暴露到公网。数据要走两层，**每一层都要同时转发 UDP 和 TCP**：

```
Teavy ──公网──> Lucky（UDP+TCP 59500）──> frps 所在电脑的 59500（UDP+TCP）
                                              │ frp 隧道
                                              └──> Mac 上的 frpc ──> 127.0.0.1:5950（Teavy Connect）
```

**第一层：frps 到 frpc。**frpc 为同一个本地端口建两个代理，一个 `tcp`、一个 `udp`，远程端口相同：

```toml
# Mac 上的 frpc.toml（frp v0.52 起的 TOML 写法）
serverAddr = "frps.example.com"
serverPort = 7001
auth.token = "换成你自己的"

[[proxies]]
name = "teavy-connect-tcp"
type = "tcp"
localIP = "127.0.0.1"
localPort = 5950
remotePort = 59500

[[proxies]]
name = "teavy-connect-udp"
type = "udp"
localIP = "127.0.0.1"
localPort = 5950
remotePort = 59500
```

frps 那边要允许这个远程端口（如果设了 `allowPorts`），frps 所在电脑的防火墙要同时放行 59500 的 UDP 和 TCP：

```toml
# frps.toml
bindPort = 7001
allowPorts = [{ single = 59500 }]
```

**第二层：Lucky 到 frps。**在 Lucky 的端口转发里给 59500 建规则，**转发类型选 TCP 和 UDP（或建两条）**，目标是 frps 所在电脑的 59500。frpc 连 frps 的 `bindPort`（上例 7001，TCP）也要能从外面到达。

外部地址填 `frps.example.com:59500`。

> **QUIC 连不上、总是退回 TCP？**转发器（Lucky 等）的 UDP 转发如果监听在通配地址，而路由器的 WAN 口有多个 IPv6 地址，回包可能从另一个地址发出，被客户端的网络丢掉。现象是 Teavy 提示“视频流不可用：UDP 不可达”、统计里显示经 Teavy Connect · TCP。把转发规则的监听地址填成域名解析到的那个地址，或者让 WAN 口只有一个 IPv6 地址。
>
> frps 自己转发 UDP 时也一样：frps 所在的电脑有多个 IPv6 地址，或者开着 sing-box、Clash 这类接管默认路由的 TUN 代理时，frps 回的包会从另一个地址（甚至 TUN 的内网地址）发出。frps 跑在一台装了 Teavy Connect 的 Mac 上时，用下面的“UDP 中转”，不用改 frps；否则可以在 frps.toml 里把 `proxyBindAddr` 设成域名解析到的那个地址（它对这个 frps 上的所有代理都生效），或者让 TUN 代理不接管这个端口的回包。

#### frps 跑在一台装了 Teavy Connect 的 Mac 上：UDP 中转

例如家里的 Mac mini 跑着 frps（它自己也装了 Teavy Connect），带出门的 MacBook 经 frpc 连回来。让家里这台 Mac 的 Teavy Connect 替 frps 收发 UDP：它在所有地址上监听，回包从收包的那个地址发出，再经本机转给 frps 的 UDP 代理口。TCP 照旧由 frps 直接收。

```
家里的 Mac：
  Teavy ──TCP 59500──> frps ───────────────────────────────┐
  Teavy ──UDP 59500──> Teavy Connect（中转）                 │
                          └──> 127.0.0.1:15950（frps 的 UDP 代理口）
                                                             │ frp 隧道
MacBook：                                                    ▼
  frpc ──> 127.0.0.1:5950（MacBook 上的 Teavy Connect，UDP 和 TCP）
```

1. **MacBook 上的 frpc**：TCP 和 UDP 映射到不同的远程端口，TCP 给客户端直连，UDP 只给中转用：

   ```toml
   [[proxies]]
   name = "teavy-connect-tcp"
   type = "tcp"
   localIP = "127.0.0.1"
   localPort = 5950
   remotePort = 59500

   [[proxies]]
   name = "teavy-connect-udp"
   type = "udp"
   localIP = "127.0.0.1"
   localPort = 5950
   remotePort = 15950
   ```

2. **家里那台 Mac 的 Teavy Connect**：设置 › 高级 › UDP 中转 › 添加中转，填 `59500 → 127.0.0.1:15950`。它只转到本机。中转的端口不能和这台 Mac 自己的 Teavy Connect 端口重。
3. **路由器或防火墙**：把 59500 的 UDP 和 TCP 都放行到家里这台 Mac（TCP 到 frps，UDP 到 Teavy Connect，同一个端口号）。15950 不要放行。
4. **MacBook 上的 Teavy Connect**：外部地址填 `home.example.com:59500`。

需要两台 Mac 上的 Teavy Connect 和 iPhone、iPad 上的 Teavy 都是支持这项功能的版本：旧版的 Teavy 会让 MacBook 发出过大的包，QUIC 连上了却没有画面。

frp 是把每个 UDP 包装进 frpc 到 frps 的那条连接（默认是 TCP 加 TLS）里转的，那条连接上丢一个包，后面的包都要等它重传，画面偶尔会卡一下。能直连（做法一、做法三）就直连。

### 做法三：IPv6 直连，不经转发器

家里有公网 IPv6 时可以不经任何转发器：路由器给这台 Mac 固定一个 IPv6 后缀，防火墙直接放行 Teavy Connect 的端口，DDNS 用前缀拼上后缀。下面以 OpenWrt 为例，Mac 的 IPv6 地址固定为 `<前缀>::5950`。

1. **固定后缀**：静态租约里给这台 Mac 设 IPv6 令牌（后缀），并让局域网的路由通告带上“管理配置（M）”标记，Mac 才会经 DHCPv6 领到 `<前缀>::5950`：

   ```
   # /etc/config/dhcp
   config host
       option name 'mac'
       option mac 'aa:bb:cc:dd:ee:ff'   # 这台 Mac 的网卡地址
       option hostid '5950'             # LuCI 静态租约里的“IPv6 令牌”

   config dhcp 'lan'
       option dhcpv6 'server'
       list ra_flags 'managed-config'   # LuCI 里 RA 标识选“管理配置 (M)”
   ```

   改完后让 Mac 重新连一下网络，在“系统设置 › 网络 › 详细信息 › TCP/IP”里能看到这个地址。

2. **防火墙放行**：从 WAN 到这台 Mac 的 5950，UDP 和 TCP 都放行，只放 IPv6。目标地址按后缀匹配，运营商换了前缀规则照样有效（LuCI 里在目标地址的“自定义”里填）：

   ```
   # /etc/config/firewall
   config rule
       option name 'Allow-Teavy-Connect'
       option src 'wan'
       option dest 'lan'
       option family 'ipv6'
       list proto 'tcp'
       list proto 'udp'
       option dest_ip '::5950/::ffff:ffff:ffff:ffff'
       option dest_port '5950'
       option target 'ACCEPT'
   ```

3. **DDNS**：用脚本取 lan 口的 /64 前缀拼上后缀，交给你的 DDNS 工具（Lucky、ddns-scripts 等）更新 AAAA 记录：

   ```sh
   prefix=$(ubus call network.interface.lan status | jsonfilter -e '@["ipv6-prefix-assignment"][0].address')
   echo "${prefix%::}::5950"   # 例如 2001:db8:1:2::5950
   ```

外部地址填 `home.example.com:5950`。这种做法只有 IPv6：客户端所在的网络（大多数蜂窝网络有）也要有 IPv6 才连得上。

## 卸载

菜单里选“卸载后台代理…”，再把 Teavy Connect 从“应用程序”里删掉。配对和代理的身份留在 `/Library/Application Support/Teavy Connect`，重新安装后照常可用；不再需要的话用管理员权限删掉这个目录。系统里留下的隐藏用户 `_teavyconnect` 删不掉（删除用户记录需要“完全磁盘访问”），不影响使用，重装时沿用。
