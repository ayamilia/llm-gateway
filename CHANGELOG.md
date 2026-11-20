# 更新日志

本项目的取法是「自用的最小化网关」：功能都从实际跑起来的线上需求长出来，**新增能力一律默认关闭**，旧 `config.json` 不需要迁移，不配就等于没有这块。下面按能力归类，不按提交顺序。

## 0.3.0 — 2026-11-20

### 修复：协议互转时图片被静默丢弃

四个互转函数（`chatToMessages`、`chatToResponses`、`messagesToChatReq`、`responsesToChatReq`）以前都把 content 压成 `textOf()`，图片块变成空字符串——**HTTP 200、日志无异常，但模型说没看见图**。现在统一走一组转换器（`imagePartOf` / `chatImagePart` / `messagesImagePart` / `responsesImagePart` / `imagesIn`）：

- 四个方向都保留图片；`data:` URL 自动拆成 base64（Anthropic 的 `source.type:"url"` 不认 data URL，实测会被拒）。
- **没有图片时仍输出纯字符串**，请求体与旧版逐字一致（prompt 前缀/缓存不受影响）。
- 同协议直通本来就逐字节转发，不涉及。

### 修复：Gemini 上游的两个硬约束

- **工具声明**：`functionDeclarations.parameters` 只吃 OpenAPI 子集，多一个 `additionalProperties` 就整条 400（`Invalid JSON payload received. Unknown name "additionalProperties"`）。新增 `geminiSchema()` 递归白名单清洗：丢 `additionalProperties`/`$schema`/`$ref`/`format`/`default`/`exclusiveMinimum`，`oneOf`+`allOf` → `anyOf`，`const` → `enum`，`required` 去掉未声明字段。Hermes/Pi 这类客户端的工具 schema 普遍带 `additionalProperties: false`，不洗的话带工具的请求全废。
- **轮次**：对话必须以 user 轮结尾且同角色不连续。带 assistant 预填充（prefill）的请求会被拒（`Requests ending with a model turn are not supported.`）——现在相邻同角色先合并 parts，末尾若是 model 就追加一轮 user（文本可配 `config.gemini_continue`，缺省「继续」）。

### 新增：供应商级默认协议（`providers.*.format`）

同一家上游的模型多半只认同一种协议，逐模型配 `model_formats` 太啰嗦。供应商可设 `chat` / `responses` / `messages` / `gemini`，留空＝自动探测。解析顺序：路由显式 > 模型显式 > 供应商显式 > 探测记忆 > 正则推断 > chat（**显式配置统一排在探测记忆前面**）。改/清该值时会顺手删掉这家供应商的探测记忆，避免旧值顶回来。面板有下拉与「协议」列。

### 变更：搜索代理默认不介入（`web_search.mode`）

以前 `enabled: true` 就对**所有上游**注入 `web_search` 工具，模型一搜即由网关代打搜索 REST——客户端从没要过搜索也会产生费用。现在默认 `off`（`enabled: true` 但没写 `mode` 也按 off），另有两档：`declared`（客户端自己声明同名工具才接管）、`inject`（旧行为，需显式配置，建议用 `providers`/`models` 收窄）。

### 性能：额度快照不再阻塞首屏

`/api/upstream/quota` 原来是「缓存过期就同步重算」，一次要串行等上游再聚合，首屏白等 1 秒以上。现在：

- **stale-while-revalidate**：过期时立刻返回旧快照，后台刷新。
- 两条独立上游调用并行；并发请求用单飞（in-flight）去重，不会各跑一份。
- 快照落盘 `state/quota-cache.json`，重启后先拿它垫首屏；启动时预热一次。
- 重算会打一行 `[quota] 重算 <ms>（上游 x · 本地 y · 每模型 z）`，便于定位。

实测：热缓存 5–11ms、过期后首屏 12ms、冷启动（无磁盘快照）530ms。

> 说明：线上部署里的「按某家模型目录自动取单价/限额」「按某家控制台拉每模型用量」依赖特定厂商接口，**未纳入开源版**；单价与月限额仍由 `config.prices` / `peak_prices` 提供。

## 0.2.0 — 2026-11-18

### 新增：联网搜索代理（`web_search`）

有些上游把联网搜索做成**自己的服务端工具**（`tools[]` 里 `type` 不是 `function`，结果也不在 `tool_calls` 里），只有那一家上游认；模型走别家中转时完全用不上。

这个功能反过来做：网关向**任意上游**声明一个普通 `function` 工具 `web_search`，模型调用它时由网关自己请求上游的搜索 REST（`<web_search.base_url>/tools/search`），把结果作为 tool 消息回灌，再续跑一轮。客户端只看到最终答案，**不需要支持任何服务端工具协议**。

- 三种客户端协议都覆盖：chat、responses、messages。同协议直通插不进循环，所以开了代理的 responses/messages 请求会自动改走 chat 中枢（日志 `WS INTEROP`）。
- 客户端自带服务端工具（`type != function`）时**完全不介入**：不注入、也不改道，保持逐字节直通。
- 客户端自带的 function 工具与代理共存；模型同一轮同时调两类时，代理调用被网关执行并续跑，客户端工具调用由模型下一轮重新决定。
- 每轮 usage 累加；搜索次数写进统计记录的 `ws_calls` 字段。
- `max_rounds`（续跑轮数）/ `max_calls`（单请求搜索次数）/ `count` / `need_content` / `timeout_ms` 均可配；`providers` / `models` 可把注入范围收窄到特定上游或模型。
- 日志：`WS SEARCH <名> "<query>" <字节>`、`WS ROUND <轮> <客户端>-><上游>`、`WS UPSTREAM ERROR`。

### 新增：搜索 / 抓取 REST 透传（`/v1/tools/search`、`/v1/tools/fetch`）

上游的独立 Search / Fetch 接口（不是模型工具，是两条 JSON API）现在可以直接经网关调用：用网关客户端 key 鉴权，body 原样进、响应原样出，记一条统计（`model=tools/search|fetch`）。`web_search.expose_tools_api: false` 可单独关掉。

### 新增：供应商按协议覆盖 base（`format_bases`）

同一家上游的不同协议不一定在同一个 base 下——典型是 Anthropic 兼容端点在 `<host>/anthropic/v1/messages`，而 `base_url` 是 `<host>/v1`，按 `base_url + /messages` 拼就是 404。

```json
"providers": { "upstream-a": {
  "base_url": "https://api.example.com/v1",
  "format_bases": { "messages": "https://api.example.com/anthropic/v1" }
} }
```

读取收敛在 `upstreamBase(provider, format)`，四个拼接点（chat / responses / messages / gemini）、原生直通、推理档位探测全部走它；不配就是原来的行为。

### 新增：旧工具名拦截（`tool_name_guard`）

服务端工具名写错时，上游往往只回一个含糊的 422，或者干脆 200 但**静默不搜**（白烧 token）。现在可以直接 400 并给出可执行的提示：

```json
"tool_name_guard": { "enabled": true, "providers": ["upstream-a"],
  "bad": ["web_search", "web_search_preview", "web_search_20250305"],
  "message": "工具名 \"{bad}\" 不被 {provider} 接受……" }
```

- 只拦「服务端工具形态」（`tools[].type != "function"`）；客户端自己的 function 工具即使叫 `web_search` 也不动。
- `providers` 缺省为空＝不拦任何供应商；写 `["*"]` 全局生效。
- 拦截点在 chat 入口与 responses/messages 入口（`routeFor` 之后、调上游之前），命中也记一条 `400 tool_name_invalid:<名>`。

### 新增：上游池 `drain` 策略（到期优先）

同厂商多包月订阅「排队烧」：按 `monthly.resetsAt` 升序排队，只用队首那只，它触门槛才换下一只，会话仍粘在同一只上。三道门槛 `drain_min_monthly` / `drain_min_weekly` / `drain_min_rolling`（默认 3 / 5 / 5，百分比）——周窗口那道是关键，月剩 20% 但周剩 0.2% 时把重会话搬过去会打到一半被限流。

配套加了组级开关 `pools.<名>.balance: false`：这个组不参与均衡，固定走 `members[0]`，只在它挂了/进冷却时换人（自己一个人用的组不该被摊平到多个账号上，会打散 prefix 缓存）。

### 新增：按会话统计

面板新增「按会话」子页，按真正发给上游的会话 id（`osid`）分组；空值与网关自动填的缺省 id 不纳入（一个 key/账号永远同一个、不区分对话的那些）。该维度整段读明细表，只在切到该页时才计算（`/api/stats?sessions=1`）。

### 新增：官方总量叠加线

概览柱状图可叠一条「官方总量」线（只在最近 5 小时 / 本周 / 本月三个口径），数据来自上游控制台的用量接口（`quota.official_usage.url`）。窗口按浏览器时区对齐，一律先拉小时桶再由服务端重聚合（官方 day 桶是 UTC 日，+08:00 下会错位 8 小时）。

### 修复与打磨

- 非流式请求在中继路径上的渲染顺序（少一个分支就会「日志 OK、客户端收不到任何字节」）。
- 被代理吞掉的服务端工具调用，不再因上游在 `.done` 里重发全量参数而重复追加（表现为多出一次空查询）。
- 续跑失败（上游 5xx）现在会把错误透给客户端，不再静默返回半截答案。
- `usage` 提取、字节统计、TTFT 等统计字段与前面几版保持一致，新增字段都向后兼容（老记录照样能聚合）。

## 0.1.0 — 2026-11-05

首个公开版本：chat 中枢 + 三种上游协议（chat / responses / messages / gemini 原生）、协议互转（按 key 开关）、上游池（`sticky` / `quota` / `rr` / `least` / `failover`）、key 模型白名单与上游组绑定、分用户鉴权、会话身份与替代 UA、推理档位探测、推理链回灌、抓包开关、首字节保活、SQLite 统计（含缓存命中、token/s、字节、调用方 IP）。
