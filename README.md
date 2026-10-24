# CF-Workers-TGProxy

Cloudflare Workers 的 Telegram Web/MTProto WebSocket 代理中继（KWS Lanes）。

[`telegram_webproxy.js`](./telegram_webproxy.js) 是一个单文件 Cloudflare Workers 实现：它扮演 Telegram 官方 Web Proxy（td web-proxy）后端，把客户端（Desktop / Android）在隔离 webview 里发起的 WebSocket lanes，直接转接到 **Telegram Web K 自己的 KWS 通道**（`kws{dc}[-1].web.telegram.org/apiws`）。

它不再依赖一套独立的 MTProxy / middle proxy 后端：Worker 在本地解开客户端 MTProto 传输层的 obfuscation，再用自己生成的随机密钥重新封一层，拨 Telegram 官方的 Web K 入口。内层 MTProto 载荷始终由客户端端到端加密，Worker 不接触、也不做 mtp 层加解密。此外，还内置了**开箱即用的落地看板与实时负载监控**，便于用户直接分发与观测。

## 核心能力

当前代码主线包含：

- **自适应服务落地看板**：浏览器直接访问时展示 Element 风格的代理服务中心，全自动提取当前域名与密钥生成 `tg://webproxy` 专属一键导入与复制链接
- **Edge 内存实时监控**：无需外部数据库或重型组件，在 Worker 内存中零开销埋点采集在线连接数、累计吞吐流量、会话总数与 Uptime，提供 `/api/stats` 接口与前端 3 秒自刷新展示
- td web-proxy 页面：nonce + 严格 CSP，仅允许 `frame-ancestors http://127.0.0.1:*`
- Android 应用内 web-proxy 桥（`globalThis.TelegramWebProxy` 双向 postMessage）
- 多 lane WebSocket 复用：子协议 `tproxy-lane-v1.<token>.<sid>`
- MTProto 传输层 obfuscation 重签（mode `0xef` / `0xee` / `0xdd`，DC `1..5`，media 标志）
- 拨号 Telegram 官方 KWS 端点，子协议 `binary`
- HMAC 签名的 bootstrap / session token，以及按会话的 lease 记账
- 上行 / 下行小包汇聚（`upPack = 20KB`、`dnPack = 32KB`、`dnMs = 1ms`）
- 显式关闭 WebSocket 压缩协商

## 代码主路径

```mermaid
flowchart LR
  subgraph Client ["客户端与访问端"]
    TG["Telegram 客户端 (Desktop / Android)"]
    Browser["普通用户浏览器"]
  end

  subgraph Worker ["Cloudflare Worker 边缘路由"]
    Dash["落地看板 HTML (含 /api/stats 轮询)"]
    P["Worker 隧道页面 (nonce CSP)"]
    WS["/api/v1/ws (Lanes 中继)"]
  end

  Browser -->|"GET / (无 bridge)"| Dash
  TG -->|"GET /?bridge=... (签名验证)"| P
  P -->|"建立 session 与 lanes"| WS
  WS -->|"解 obfuscation 并 mkHead 重签"| KWS["Telegram Web K / KWS 官方通道"]
```

```text
1. 普通浏览器访问：
  GET / (无签名) → 动态下发服务看板（展示 tg://webproxy 导入链接 + 3秒轮询 /api/stats）

2. Telegram 客户端连接：
  webview 页面
    → 建立 /api/v1/session（POST，带 bootstrap）
    → 拿到 session token + X-Carrier-Mode: websocket-lanes
    → 按流开 lane：/api/v1/ws + 子协议 tproxy-lane-v1.<token>.<sid>
    → Worker: 解 obfuscation → 重新封头 → 拨 KWS → 中继（同步累加流量与连接指标）
```

```markdown
# CF-Workers-TGProxy

Cloudflare Workers 的 Telegram Web/MTProto WebSocket 代理中继（KWS Lanes）。

[`telegram_webproxy.js`](./telegram_webproxy.js) 是一个单文件 Cloudflare Workers 实现：它扮演 Telegram 官方 Web Proxy（td web-proxy）后端，把客户端（Desktop / Android）在隔离 webview 里发起的 WebSocket lanes，直接转接到 **Telegram Web K 自己的 KWS 通道**（`kws{dc}[-1].web.telegram.org/apiws`）。

它不再依赖一套独立的 MTProxy / middle proxy 后端：Worker 在本地解开客户端 MTProto 传输层的 obfuscation，再用自己生成的随机密钥重新封一层，拨 Telegram 官方的 Web K 入口。内层 MTProto 载荷始终由客户端端到端加密，Worker 不接触、也不做 mtp 层加解密。此外，还内置了**开箱即用的落地看板与实时负载监控**，便于用户直接分发与观测。

## 核心能力

当前代码主线包含：

- **自适应服务落地看板**：浏览器直接访问时展示 Element 风格的代理服务中心，全自动提取当前域名与密钥生成 `tg://webproxy` 专属一键导入与复制链接
- **Edge 内存实时监控**：无需外部数据库或重型组件，在 Worker 内存中零开销埋点采集在线连接数、累计吞吐流量、会话总数与 Uptime，提供 `/api/stats` 接口与前端 3 秒自刷新展示
- td web-proxy 页面：nonce + 严格 CSP，仅允许 `frame-ancestors http://127.0.0.1:*`
- Android 应用内 web-proxy 桥（`globalThis.TelegramWebProxy` 双向 postMessage）
- 多 lane WebSocket 复用：子协议 `tproxy-lane-v1.<token>.<sid>`
- MTProto 传输层 obfuscation 重签（mode `0xef` / `0xee` / `0xdd`，DC `1..5`，media 标志）
- 拨号 Telegram 官方 KWS 端点，子协议 `binary`
- HMAC 签名的 bootstrap / session token，以及按会话的 lease 记账
- 上行 / 下行小包汇聚（`upPack = 20KB`、`dnPack = 32KB`、`dnMs = 1ms`）
- 显式关闭 WebSocket 压缩协商

## 代码主路径

```mermaid
flowchart LR
  subgraph 客户端与普通访问
    TG["Telegram 客户端 (Desktop / Android)"]
    Browser["普通用户浏览器"]
  end

  subgraph Cloudflare Worker 边缘路由
    Route{"路由与请求判定"}
    Dash["落地看板 HTML (/api/stats 轮询)"]
    P["Worker 隧道页面 (nonce CSP)"]
    WS["/api/v1/ws (Lanes 中继)"]
  end

  Browser -->|"GET / (无 bridge)"| Dash
  TG -->|"GET /?bridge=... (签名校验)"| P
  P -->|"建立 session 与 lanes"| WS
  WS -->|"解 obfuscation (AES-CTR) -> mkHead 重签"| KWS["Telegram Web K / KWS 官方通道"]
```

```text
1. 普通浏览器访问：
  GET / (无签名) → 动态下发服务看板（展示 tg://webproxy 导入链接 + 3秒轮询 /api/stats）

2. Telegram 客户端连接：
  webview 页面
    → 建立 /api/v1/session（POST，带 bootstrap）
    → 拿到 session token + X-Carrier-Mode: websocket-lanes
    → 按流开 lane：/api/v1/ws + 子协议 tproxy-lane-v1.<token>.<sid>
    → Worker: 解 obfuscation → 重新封头 → 拨 KWS → 中继（同步累加流量与连接指标）
```

## 为什么从官方 MTProto 转向 KWS 转换

Telegram 官方的 web-proxy 模型（tproxy）需要一套**独立后端**：客户端连到本地/自建的服务，再经它转发到远端 MTProxy，链路里多一跳、多一套需要自己运维的加解密中转。

本实现把这一整套换掉：

| 维度 | 经 middle/MTProxy 的官方 web-proxy 路径 | 本实现（KWS 转换） |
| --- | --- | --- |
| 后端 | 需自建 MTProxy / middle proxy | 无，Worker 即后端 |
| 传输面加密 | 代理侧承担 mtp 层与 obfuscation | 只落一层 obfuscation（AES-CTR）重签 |
| 出站目标 | 自建中转服务器 | Telegram 官方 `kws*.web.telegram.org` |
| 内层 MTProto 载荷 | 端到端（客户端） | 端到端（客户端，Worker 不接触） |

也就是说，转向 KWS 的关键点是：**Worker 只处理传输层，不处理 mtp 层**。客户端 MTProto 载荷原样透传，Worker 只对 64 字节 obfuscation 头做一次解、一次重签，再走 Telegram Web K 用的同一入口，边缘可达性与线路质量都直接吃官方域名。

## 性能理论上提升多少倍

> 以下为基于当前代码路径的**理论模型**（标记 `[推断]`），不是本仓库的实测基准。

- **数据面加解密量下降**：相对"经 middle/MTProxy 转发"的路径，本实现每包只做一次 obfuscation（AES-CTR，128-bit 计数器）重签，而不是在代理侧再承担 mtp 层的加解密。数据面 AES 块操作量约降到一半量级 `[推断]`。
- **少一跳 RTT**：去掉自建中转服务器，客户端 → CF 边缘 → Telegram KWS。`[推断]`
- **小包汇聚**：上行 `upPack = 20KB`、下行 `dnPack = 32KB` + `1ms` 观察窗，把高频 tiny frame 压成更少的实际写入。复用 GrainTCP 同一颗 grain 核，其本地回放实测区间为 **1.8x–39.8x**（固定 512B 风暴下约 39.8x，mixed 小包约 1.8–2.0x）。
- **单 isolate 承载**：单会话最多 128 条 lane，无需把整条会话钉在单个 Durable Object 上。

综合到瓶颈场景，理论提升从 **约 2x 起**（仅算加解密 + 去跳），小包风暴场景在汇聚部分叠加后可更高。`[推断]`

## 设计重点

### 1. 多 lane 直接替代 Durable Object

公开的 Workers web-proxy 实现大多用 Durable Object 来维持"单条长连接 / 单份状态"。DO 是单线程且需要路由到固定实例，上限和排队都集中在那个对象上。

本实现改为**按流拆 lane**：

- 每条 Telegram 流 = 一条独立 WebSocket，子协议里带 `sid`
- 所有 lane 仍落在**同一个 isolate**，session / lease 状态是 isolate 内的 `Map`
- 单 lane 上限：`laneMax = 8MB` / `laneItems = 1024`
- 整会话队列上限：`qMax = 32MB` / `iMax = 16384`
- lane 数上限 `128`，已用 sid 上限 `4096`

TG 客户端本身就有多流、并且会自动分流，所以 lanes 让**每条流各自背压、各自排队**，而不是把整条会话压进一个单线程对象。这正是"不需要 DO、TG 侧也感觉不到单连接限流"的设计依据。

### 2. 小包汇聚

上下行各自把连续小块先收进一颗薄核，再尽量并成更少的实际写入：

```text
上传: collect -> bundle -> peer.write()
下载: collect -> bundle -> dnPack 门控 -> ws.send()
```

- `upPack = 20KB`：上行单次合包目标
- `dnPack = 32KB`：下行聚合上限；`>= 32KB` 直接发，`< 32KB` 进核再等门控
- `dnMs = 1ms`：下行 quiet-window，用来决定何时 flush

目的都是削减高频小 `frame` 带来的固定调度成本，而不是再造一层重型队列。

### 3. 落地看板与轻量级实时监控

无需额外搭建管理端或引入重型数据库：
- **动静分流识别**：请求根路径 `/` 时，若携带 Telegram Desktop / Android 校验签名的 `?bridge=` 参数，严格走带有严格 CSP 的隧道通道 `mkPage`；若为普通浏览器直接访问，则下发美观的 Element 风格落地看板。
- **动态配置适配**：看板自动读取用户当前访问的 Hostname 及环境绑定的 `SECRET`，动态拼接 `tg://webproxy?server=${host}&secret=${secret}`，免除部署后手工修改前端配置的繁琐。
- **零开销内存埋点**：
  - **实时在线人数**：在 WebSocket 握手成功与断开关闭回调（`onClose`）中同步对 `activeConnections` 计数。
  - **吞吐与会话**：在 `write()`（上行）与 `emitDn()`（下行）处直读缓冲区分片大小累加至 `totalTrafficBytes`，在 `/api/v1/session` 建立时统计总会话。
  - 前端基于原生 JS 每 3 秒轻量轮询 `/api/stats`，并实现带降级兜底的原生剪贴板复制提示。

### 4. 鉴权与租约

- 页面只下发一次性 `nonce` + 短期 `bootstrap`（`mkBoot`，2 分钟）
- `POST /api/v1/session` 校验 bootstrap 后签发 `session` token（`mkSess`，5 分钟）
- 每个 session token 对应一份 `lease`，记录 `active` / `used` sid、过期与删除状态
- `DELETE /api/v1/session` 置 `lease.deleted`，后续 lane 直接 `409`/关闭
- token 为 HMAC-SHA256 截断签名，签名/比较走常量时间

### 5. 页面隔离

- 客户端隧道页面 CSP 采用 `default-src 'none'`、`sandbox allow-same-origin allow-scripts`、`frame-ancestors http://127.0.0.1:*`
- `/api/v1/ws` 若带 `Origin` 头，必须与自身同源，否则拒绝

### 6. 禁用 WebSocket 压缩协商

握手响应里显式把 `Sec-WebSocket-Extensions` 置空。relay 主线传的是原始 TCP 字节流（内部多为 TLS/HTTP2 等高熵数据），在 Worker 这层再做 WS 压缩换不来稳定收益，反而抬高热路径 CPU，因此按负优化处理。

## 当前配置

| 变量 | 意义 | 默认值 |
| --- | --- | --- |
| `dnPack` | 下行聚合上限 | `32 * 1024` |
| `upPack` | 上行合包目标 | `20 * 1024` |
| `dnMs` | 下行 quiet-window | `1`（ms） |
| lane 上限 | 单会话并发 lane 数 | `128` |
| `laneMax` / `laneItems` | 单 lane 字节 / 条数上限 | `8MB` / `1024` |
| `qMax` / `iMax` | 整会话字节 / 条数上限 | `32MB` / `16384` |
| bootstrap TTL | 一次性接入 token | `2` 分钟 |
| session TTL | 会话 token | `5` 分钟 |
| KWS 端点 | Telegram Web K 出站 | `kws{1..5}[-1].web.telegram.org/apiws` |

## 密钥与环境变量配置

### 必需变量

| 绑定 | 意义 | 要求 |
| --- | --- | --- |
| `SECRET` | 代理密钥：AES-CTR 传输密钥派生 + `?bridge=` 能力签名 | `32` 位 hex，或 `dd` / `ee` / `ef` 前缀 + 32 位 hex（共 `34`）；字符集 `0-9a-f`；大小写不敏感，建议小写 |
| `BOOT` | bootstrap token：签发 bootstrap / session，并内联进下发页面 | 仅要求非空，建议 ≥ `16` 字符随机串 |

生成命令：

| 目标 | 命令 | 输出 |
| --- | --- | --- |
| `SECRET` | `openssl rand -hex 16` | `32` 位 hex |
| `SECRET`（带前缀） | `printf 'dd%s\n' "$(openssl rand -hex 16)"` | `34` 位 hex |
| `BOOT` | `openssl rand -hex 24` | `48` 位 hex |

`openssl rand` 走系统 CSPRNG，是真随机。任一必需绑定缺失，`mkCfg` 直接抛错，`/health` 会返回 `500` 而不是 `{"ok":true}`。

### 可选变量（自定义看板）

| 绑定 | 意义 | 说明 | 默认值 |
| --- | --- | --- | --- |
| `TG_BOT` | 交流群组 / 机器人用户名 | 看板底部联系方式按钮所指向的 TG 机器人用户名（无需加 `@`） | `eoops_bot` |
| `MACHINE_BOOT_TIME` | 边缘节点/机器启动时间 | 自定义在看板指标卡中展示的系统启动时间字符串 | Worker 实例初始化时间 |
| `MACHINE_UPTIME` | 累计运行时长文本 | 自定义在看板指标卡中展示的机器累计运行时间 | 实例存活计算时长 |

## 路由

| 路径 | 方法 | 说明 |
| --- | --- | --- |
| `/` | `GET` / `HEAD` | **带 bridge 签名**：下发内置 web-proxy 通信隧道页面<br>**浏览器直接访问**：下发代理服务中心落地看板（含一键导入链接与实时指标） |
| `/api/stats` | `GET` | 实时监控接口，返回当前在线人数、吞吐、会话数与 Uptime 的 JSON 数据 |
| `/health` | `GET` | 节点健康检查，返回 `{"ok":true}` |
| `/api/v1/session` | `POST` | 建会话，返回 session token（累计会话计数 +1） |
| `/api/v1/session` | `DELETE` | 注销会话 |
| `/api/v1/ws` | `GET` | lane WebSocket（需 `Upgrade` 与正确子协议，建立/销毁时维护在线并发计数） |

## 文件

| 文件 | 说明 |
| --- | --- |
| [telegram_webproxy.js](./telegram_webproxy.js) | Worker 主实现：页面 / 监控仪表盘 / session / lane / obfuscation / KWS 中继 |

## 访问优化（CNAME / SaaS 优选）

- 使用**自有域名 + CNAME**，或直接走 Cloudflare **SaaS（自定义主机名）优选**，以获得更好的接入体验。
- 参考实现中曾出现第三方线路（非官方 `web.telegram.org` 域名）导致边缘层 `1034` / `403`；换成官方 web 域名后恢复正常，因此**优先使用 Telegram 官方 web 域名与稳定的自有 CNAME / Custom Domain**，避免依赖质量不稳定的中间线路。

## 相关链接

- 开源协议：[GPL-3.0](./LICENSE)
- 频道 / 交流群组：<https://t.me/Enkelte_notif>
- Telegram Desktop（tdesktop）：<https://github.com/telegramdesktop/tdesktop>
- Telegram Android（DrKLO）：<https://github.com/DrKLO/Telegram>
- Telegram tproxy-server：<https://github.com/telegramdesktop/tproxy-server>

## Stargazers over time

<a href="https://www.star-history.com/?repos=toicf%2Fcf-workers-tgproxy&type=date&legend=top-left">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=toicf/cf-workers-tgproxy&type=date&theme=dark&legend=top-left" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=toicf/cf-workers-tgproxy&type=date&legend=top-left" />
   <img alt="Star History Chart" src="https://api.star-history.com/chart?repos=toicf/cf-workers-tgproxy&type=date&legend=top-left" />
 </picture>
</a>

## 友情链接

- [linux.do——](https://linux.do/)一个蓬勃发展的开发者社区。
