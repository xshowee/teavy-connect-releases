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
- **不要再把屏幕共享的 5900 端口暴露到公网。**以前为屏幕共享做的 5900 端口映射或内网穿透，有了 Teavy Connect 之后就删掉，从外面只转发 Teavy Connect 的端口。
- 在 Teavy Connect 的设置里把“外部地址”填成从外面到达这个端口的域名加端口（例如 `home.example.com:5950`），配对时 Teavy 会记下它。

下面的域名、地址和端口都是示例，换成你自己的。

### 做法一：路由器端口映射（有公网 IPv4）

在路由器上把公网的 5950 映射到这台 Mac 的局域网地址的 5950，**协议选 TCP 和 UDP 两项**（有的路由器要分两条规则建）。域名用路由器或 DDNS 服务维护。

### 做法二：IPv6 直连

家里有公网 IPv6 时可以不经任何转发：

1. 让路由器给这台 Mac 一个固定的 IPv6 地址（例如用 DHCPv6 静态租约固定后缀）。
2. 防火墙放行从外网到这个地址的 5950，UDP 和 TCP 都放。按后缀匹配目标地址的话，运营商换了前缀规则照样有效。
3. 用 DDNS 维护一条指向这个地址的 AAAA 记录。

外部地址填 `home.example.com:5950`。这种做法只有 IPv6：客户端所在的网络（大多数蜂窝网络有）也要有 IPv6 才连得上。

### 做法三：内网穿透

适合 Mac 不在家里（比如随身带出门的 MacBook）：Mac 上的穿透客户端连回一台公网可达的服务端（家里的电脑、路由器或云服务器），由服务端把端口暴露出去。

- **每一层都要同时转发 UDP 和 TCP。**穿透配置里为 5950 建 UDP 和 TCP 两条映射，服务端前面如果还有一层端口转发，也要两种协议都转。穿透客户端连服务端用的那个端口也要能从外面到达。
- **QUIC 连不上、总是退回 TCP？**现象是 Teavy 提示“视频流不可用：UDP 不可达”，统计里显示经 Teavy Connect · TCP。常见原因是转发 UDP 的那一环回包用错了源地址，被客户端的网络丢掉：它监听在通配地址，而所在的机器有多个 IPv6 地址，或者开着接管默认路由的 TUN 代理。把监听地址固定成域名解析到的那个地址，或者用下面的“UDP 中转”。
- **经内网穿透时画面会慢一些。**很多内网穿透工具把 UDP 包装进一条 TCP 连接里转，丢一个包后面的都要等；往返一长，Mac 发出的码率也压不住。能直连（做法一、做法二）就直连。

#### 穿透服务端跑在一台装了 Teavy Connect 的 Mac 上：UDP 中转

例如家里的 Mac mini 跑着穿透服务端（它自己也装了 Teavy Connect），带出门的 MacBook 连回来。让家里这台 Mac 的 Teavy Connect 替穿透服务端收发 UDP：它在所有地址上监听，回包从收包的那个地址发出，再经本机转给穿透的 UDP 端口。TCP 照旧由穿透服务端直接收。

```
家里的 Mac：
  Teavy ──TCP 59500──> 穿透服务端 ─────────────────────────┐
  Teavy ──UDP 59500──> Teavy Connect（中转）                 │
                          └──> 127.0.0.1:15950（穿透的 UDP 端口）
                                                             │ 内网穿透
MacBook：                                                    ▼
  穿透客户端 ──> 127.0.0.1:5950（MacBook 上的 Teavy Connect，UDP 和 TCP）
```

1. **MacBook 上的穿透客户端**：TCP 和 UDP 映射到不同的远程端口，TCP 映射到 59500 给客户端直连，UDP 映射到 15950，只给中转用。
2. **家里那台 Mac 的 Teavy Connect**：设置 › 高级 › UDP 中转 › 添加中转，填 `59500 → 127.0.0.1:15950`。它只转到本机。中转的端口不能和这台 Mac 自己的 Teavy Connect 端口重。
3. **路由器或防火墙**：把 59500 的 UDP 和 TCP 都放行到家里这台 Mac（TCP 到穿透服务端，UDP 到 Teavy Connect，同一个端口号）。15950 不要放行。
4. **MacBook 上的 Teavy Connect**：外部地址填 `home.example.com:59500`。

需要两台 Mac 上的 Teavy Connect 和 iPhone、iPad 上的 Teavy 都是支持这项功能的版本：旧版的 Teavy 会让 MacBook 发出过大的包，QUIC 连上了却没有画面。

## 卸载

菜单里选“卸载后台代理…”，再把 Teavy Connect 从“应用程序”里删掉。配对和代理的身份留在 `/Library/Application Support/Teavy Connect`，重新安装后照常可用；不再需要的话用管理员权限删掉这个目录。系统里留下的隐藏用户 `_teavyconnect` 删不掉（删除用户记录需要“完全磁盘访问”），不影响使用，重装时沿用。
