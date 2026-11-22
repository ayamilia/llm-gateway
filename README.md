# 一个自用的最小化 LLM 网关

> 客户端只会说一种api，上游各有各的协议。网关夹在中间：对外一套接口，对内按模型名分流。

更新内容见 [CHANGELOG.md](CHANGELOG.md)。

零依赖，只用 Node 内置模块（`http` / `sqlite` / `crypto` / `zlib`）；一个 `server.mjs` + 一个 `dashboard.html`。需要 Node ≥ 22.5（用了内置 `node:sqlite`）。

## 它做什么

客户端说一种协议，上游说另一种，网关负责把它们对上：

| 客户端 → ＼ 上游 ↓ | chat | responses | messages |
|---|---|---|---|
| **chat**（`/v1/chat/completions`） | ✅ 原样透传 | ✅ 双向翻译 | ✅ 双向翻译 |
| **responses**（`/v1/responses`） | ⚙️ 可选 | ✅ 原样直通 | ⚙️ 可选 |
| **messages**（`/v1/messages`） | ⚙️ 可选 | ⚙️ 可选 | ✅ 原样直通 |

**chat 是中枢**：客户端只要说 chat，就能用任意协议的上游。responses / messages 原生客户端（Codex、Claude Code 这类）默认只能同协议直通；`⚙️ 可选` 那几格要开「协议互转」——**默认关**，见下。

| 能力 | 说明 |
|---|---|
| 路由 | 按模型名前缀或正则分流到不同上游，可 `strip` 掉前缀 |
| 三协议 | 默认自动试，成功后记忆协议类型 |
| 协议互转（可选） | **默认关**，按 key 勾选：让 responses / messages 客户端跨协议用其它协议的模型。会丢保真度，见下 |
| 推理档位 | 每个对外模型可声明「上游支持的档位」（`model_efforts`）+ 厂商自有映射（`model_effort_overrides`，如 medium→high）；客户端给的档位超范围时就近降级（只降不升），没配置＝原样透传。面板可逐档打最小请求探测，结果缓存到 `effort_probes` |
| key 白名单 | 每个客户端 key 可限定可用模型（`keys[].models`），空＝不限制；按对外模型名或「上游实体」比对，越权直接 403/404 |
| 虚拟 key | 每个客户端一把，可选绑定协议、可吊销、单独统计 |
| 上游池（可选） | 一个对外模型背后挂多个上游：`strategy` 可选 `sticky`（按客户端 key 粘性）/ `quota`（按剩余额度分流）/ `rr`（加权轮询）/ `least`（按并发）/ `failover`（恒首选）。失败自动换成员，连续失败进冷却。成员可用 `models` 声明自己提供哪些模型 |
| 会话身份 | `session_hosts` 里列的上游自动带 `x-upstream-session: __SESSION__`（运行时解析成该 key 的 `session_id`），同时给 key 预置一个替代 UA——缺这两样的上游会回 400 `MissingSessionID` 或 403 `error 1010` |
| 多人共用（可选） | 管理员在面板建账号后，普通用户只能看到自己的上游、自己的 key、自己 key 的用量；组可设 `user_join` 让别人把自己的上游加进来，模型与额度在组内共享 |
| 统计 | 每条请求落 SQLite：token / 缓存命中 / TTFT / 耗时 / 生成速度（token/s）/ 字节 / 调用方 IP / 状态 |
| 统计范围 | 面板「供应商」页可把某个上游设为「不参与统计」：它的流量不计入概览 / 按模型 / 按上游（「最近请求」始终全量）。状态只存浏览器本地，不改配置、不改数据库；默认把 base_url 指向本机（mock/测试上游）的算作不参与 |
| 面板 | 单页 HTML、无构建：用量统计（多时间口径）、模型列表、加供应商、客户端 key；暗亮主题、手机适配 |
| 供应商管理 | 「加供应商」页把上游分成**正式 / 测试**两组（可折叠，测试组默认折叠）：显式 `providers.<名>.test` 优先，没写则按 base_url 是否指向本机推断，点 tag 即可切换。加模型用**供应商下拉**选目标，接口是幂等 upsert——模型已存在就用新配置覆盖（协议留空＝清掉指定与探测缓存、回到自动） |
| 抓包开关 | 一键把「客户端原始请求体」和「翻译后发给上游的 payload」并排落盘——判锅用 |
| 首字节保活 | 上游迟迟不吐数据时，先开 SSE 流并每 10s 发注释心跳，绕开反代的 120s 超时 |
| 订阅周期用量 | 按上游用量接口给的续费时刻切「月付周期」，逐周期列出 token（输入 / 命中缓存 / 输出）、已产生费用、以及费用占该模型月限额的比例。分母基数 `quota.base_usd`（默认 60）、无接口时按 `quota.cycle_day` 切分 |
| 推理链回灌 | 把 reasoning item 按 `call_id` 归档并插回下一轮，多轮工具调用不再丢思维链 |
| 联网搜索（可选） | 网关向上游声明一个 `function` 工具 `web_search`，模型一调用就由网关去打搜索 REST、回灌结果再续跑——客户端不用支持任何服务端工具协议。**默认关**，见下 |
| 搜索 REST 透传 | 上游的独立 Search / Fetch 接口挂成 `/v1/tools/search`、`/v1/tools/fetch`，用网关 key 鉴权、原样透传 |
| 协议 base 覆盖 | 供应商可配 `format_bases`：某一种协议的上游 base 与 `base_url` 不同时（典型：Anthropic 兼容端点在 `/anthropic/v1`）单独指定，不配就是老行为 |
| 工具名守卫（可选） | 服务端工具名写错（`web_search` / `web_search_preview` 这类旧名）时直接 400 并给出正确写法，不再让上游 422 或静默不搜 |

## 协议互转（可选，默认关）

> 留着这个功能是因为「总觉得应该有」，但它本质是**围绕 chat 做的中枢式转换**，不是无损互通。除非你真的需要「messages 客户端跑 responses 上游的模型」这种组合，否则别开。

**打开**：面板「客户端 key」页 → 每个 key 一列「协议互转」勾选框（默认关）。开的是**那一个 key**，其它 key 不受影响；对应 `config.json` 里 `keys[].protocol_interop`。

| | |
|---|---|
| 做什么 | 上游协议 ≠ 客户端协议时走 `客户端协议 → chat → 上游协议`，响应再翻回来。请求侧 `responsesToChatReq` / `messagesToChatReq`；响应侧按客户端协议渲染 Responses / Anthropic 事件流（`responsesSink` / `messagesSink`） |
| 同协议 | **不受影响**，仍走原生直通（逐字节回吐，保真度最高） |
| 保真度代价 | ① Responses 的加密 reasoning item 在 chat 中枢里装不下 → 丢；② Anthropic 流式的 `message_start.usage.input_tokens` 只能给 0（真值补在 `message_delta`）。**图片块四个方向都会保留**（chat `image_url` / messages `image` / responses `input_image` 互转，data URL 自动拆成 base64）；无图时请求体与旧版逐字一致 |
| 模型可见性 | `/v1/models` 默认仍按客户端协议过滤；置 `config.json` 的 `model_filter_protocol: false` 可放开成全量 |
| 排查 | 互转请求打 `CONVERT <客户端协议> -> <上游协议>`；统计表新增 `cproto` 字段（客户端协议） |
| 回归 | `scripts/interop_test/run.sh` —— 起假上游 + 第二个网关实例，跑 3 客户端 × 3 上游 × 流式/非流式 + 多轮工具回传 |

改这段代码时**必须守住**：Responses 事件顺序里 `response.content_part.added` 不能漏——漏了 Codex 会报 `OutputTextDelta without active item` 并**丢弃全部正文**，而 HTTP 仍是 200、流也合法，最难查。

## 上游池（可选）

一个对外模型背后可以挂多个上游（多个订阅、主备、中转），路由里写 `"pool": "<池名>"` 代替 `"provider"`：

```json
"pools": {
  "pool-a": {
    "strategy": "sticky",          // sticky | quota | rr | least | failover
    "cooldown_sec": 60, "max_fails": 2,
    "members": [
      { "provider": "upstream-a", "weight": 3 },
      { "provider": "upstream-b", "weight": 1, "models": ["some-model"] }
    ]
  }
}
```

- **sticky**：按客户端 key 的 `session_id` 做稳定哈希 → 同一个 key 恒定落同一个成员，prompt 缓存与推理链不会被拆散（默认）。
- **quota**：按成员「剩余额度百分比」分流，并对每个 key 记住当前成员，只有落后超过 `quota_margin` 个百分点（默认 2）才换家——避免每轮换人把缓存废掉。
- 失败（408/429/5xx/网络错）自动换下一个成员，`max_fails` 次后进 `cooldown_sec` 冷却；全挂时回结构化错误 `upstream_pool_exhausted`，`error.attempts` 里列出每个成员的状态。
- 池成员供应商的现有路由会被自动接管，所以「同一个上游既要能单独直连、又要参与池」只用配一次。

## 多人共用（可选）

`users` 里建账号后，面板就分管理员 / 普通用户两种身份：

- 普通用户只看得到**自己的**上游、自己的 key、自己 key 的用量（概览 / 按模型 / 最近请求三张视图都按 `keys[].owner` 过滤，筛选下拉里的模型与 IP 也只来自自己的记录）。
- 池上开 `"user_join": true`，普通用户就能把自己的上游填进来一起用；退出时会连带清理自己那份供应商 / 路由 / 模型。
- 管理员在「用户」页建账号、重置密码、看每个人的加入情况。

## 会话身份与替代 UA

有的上游要求请求带一个稳定的会话标识（否则直接 400），有的会按 UA 判断客户端（非浏览器 UA 被 WAF 拦 403）。做法：

- 每个客户端 key 自带一个稳定的 `session_id`；把需要会话头的上游域名写进 `session_hosts`，网关自动给这些上游补 `x-upstream-session: __SESSION__`，值就是该 key 的 `session_id`。
- 建 key 时默认写入一个替代 UA（`keys[].user_agent`），请求时覆盖供应商的 UA；key 上留空则回落到 `providers.<n>.user_agent`。
- 想对某个上游手工指定头，直接在 `providers.<n>.extra_headers` 里写 `{"x-upstream-session": "__SESSION__"}` 或固定字符串。

## 联网搜索（可选，默认关）

> 解决的问题：有些上游把联网搜索做成**它自己上游的服务端工具**（`tools[]` 里 `type` 不是 `function`，结果也不在 `tool_calls` 里），只有那一家认；模型走别家中转时完全用不上。

做法反过来：网关向**任意上游**声明一个普通 `function` 工具 `web_search`，模型调用它时由网关自己去打搜索 REST，把结果作为 tool 消息回灌，再续跑一轮。客户端只看到最终答案——**不需要支持任何服务端工具协议**。

```json
"web_search": {
  "enabled": true,
  "mode": "declared",                          // off（默认）/ declared / inject，见下
  "base_url": "https://api.example.com/v1",   // 搜索 REST 挂在它下面的 /tools/search、/tools/fetch
  "api_key_env": "SEARCH_API_KEY",
  "tool_name": "web_search",
  "count": 5,            // 每次搜索取几条
  "need_content": false, // 是否同时抓正文（更慢、更占上下文）
  "max_rounds": 2,       // 续跑轮数（不是搜索次数）
  "max_calls": 6,        // 单请求搜索次数硬上限
  "providers": [],       // 留空＝对所有上游注入（**别留空**，见 mode）
  "models": []           // 留空＝对所有模型注入
}
```

**`mode` 决定网关介不介入**（2026-11 起默认 `off`）：

| 值 | 行为 |
|---|---|
| `off`（默认） | 不注入、不拦截、不改道。`enabled: true` 但没写 `mode` 也按 off——避免「客户端没要搜索却产生搜索计费」 |
| `declared` | 只有客户端自己在 `tools` 里声明了同名 `function` 工具才接管执行（日志 `WS TAKEOVER`）；没声明就不介入 |
| `inject` | 主动向所有（或 `providers`/`models` 限定的）上游注入 `web_search` 工具，旧行为。要用就**配 `providers`/`models` 收窄** |

> 计费提醒：`inject` 模式下模型随时可能调用，网关会代它打搜索 REST。上游按次计费的话，这就是一笔你没预料到的支出。

| 项 | 说明 |
|---|---|
| 三种客户端协议 | chat / responses / messages 都能用。同协议直通插不进循环，所以开了代理的 responses / messages 请求会自动改走 chat 中枢（日志 `WS INTEROP`） |
| 不介入的情况 | 客户端自己带了**服务端工具**（`type != "function"`）时完全不注入、也不改道，保持逐字节直通 |
| 与客户端工具共存 | 客户端自己的 function 工具照常工作；模型同一轮里两类都调时，代理调用由网关执行并续跑，客户端工具调用由模型下一轮重新决定 |
| 日志 | `WS SEARCH <名> "<query>" <字节>`、`WS ROUND <轮> <客户端>-><上游>`、`WS UPSTREAM ERROR` |
| 关掉 | `"enabled": false`；`max_rounds <= 0` 也视为关（否则会吞掉调用却不续跑） |

上游如果本来就有独立的 Search / Fetch REST，同一份 `base_url` / `api_key_env` 还会把 `/v1/tools/search`、`/v1/tools/fetch` 挂出来（用网关 key 鉴权、body 原样进、响应原样出）。不想要这两条就设 `"expose_tools_api": false`。

## 工具名守卫（可选，默认不拦任何供应商）

服务端工具名写错时，上游往往只回一个含糊的 422，或者干脆 200 但**静默不搜**（白烧 token）。开这个守卫后直接 400：

```json
"tool_name_guard": {
  "enabled": true,
  "providers": ["upstream-a"],        // 留空＝不拦；["*"]＝全局
  "bad": ["web_search", "web_search_preview", "web_search_20250305"],
  "message": "工具名 \"{bad}\" 不被 {provider} 接受，请查上游文档确认正确名字。"
}
```

- 只拦「服务端工具形态」（`tools[].type != "function"`）：客户端自己的 function 工具即使叫 `web_search` 也不会被拦。
- 拦截点在 chat 入口与 responses / messages 入口（`routeFor()` 之后、调上游之前），命中也记一条 `400 tool_name_invalid:<名>`。

## 供应商按协议换 base（`format_bases`）

同一家上游的不同协议不一定在同一个 base 下——典型是 Anthropic 兼容端点在 `<host>/anthropic/v1/messages`，而 `base_url` 是 `<host>/v1`，按 `base_url + /messages` 拼就 404：

```json
"providers": { "upstream-a": {
  "base_url": "https://api.example.com/v1",
  "format_bases": { "messages": "https://api.example.com/anthropic/v1" }
} }
```

四个拼接点（chat / responses / messages / gemini）、原生直通、档位探测全部走同一个 `upstreamBase(provider, format)`；不配就是原来的行为。

## 供应商级默认协议（`providers.*.format`）

同一家上游的模型通常只认同一种协议（例如中转站只给 Claude 开 Anthropic `/messages`），逐模型填 `model_formats` 太啰嗦。供应商也能定一档默认：

```json
"providers": { "upstream-a": {
  "base_url": "https://api.example.com/v1",
  "api_key_env": "UPSTREAM_A_KEY",
  "format": "messages"          // chat / responses / messages / gemini；留空＝自动探测
} }
```

解析顺序：**路由显式 > 模型显式（`model_formats`）> 供应商显式（`format`）> 探测记忆 > `responses_regex` 推断 > chat**。显式配置统一排在探测记忆前面，手填了就听你的。

- 改/清这个值时网关会**一并删掉该供应商的探测记忆**（`stats/format_overrides.json` 里 `供应商|*` 的条目）——否则旧记忆会把新值顶回去，表现就是「改完不生效」。
- 面板「增加供应商」里有「上游协议」下拉，供应商表多一列「协议」（未设显示 `自动`）。
- 接口：`POST /api/config/provider`（新建可带 `format`）、`/api/config/provider/update`（`format: ""` ＝ 清除）、`/api/config/provider/detail` 回显；非法值直接 400。
- 改了立刻生效，不用重启。

## 跑起来

```bash
cp config.example.json config.json    # 填 providers / routes / models / keys
cp .env.example .env                  # 填上游 API key（变量名对应 config 里的 api_key_env）
node server.mjs                       # 或 docker compose up -d
```

默认监听 `0.0.0.0:8010`，面板就是 `/`。客户端把 base_url 指到 `http://<host>:8010/v1`，key 填网关签发的虚拟 key。

配置在 `config.json`（providers / pools / routes / models / keys / users / prices / quota / session_hosts），key 在 `.env`（启动时读进进程环境，所以 `docker restart` 不会丢），统计在 `stats/requests.db`（自动建，jsonl 只滚动留 3 天）。面板上改配置会自动备份 + 热加载。

## 注意

- 抓包文件是**完整对话历史**，排查完记得关。
- 上游额度卡片默认是空的（`config.quota` 不填就不显示）；返回形状各家不同，改 `upstreamBalance()` / `upstreamUsage()` 两个函数。周期统计的锚点取 `usage.monthly.resets_at`，取不到退回 `quota.cycle_day`（默认 1 号）。
- 单价表在 `config.prices` / `config.peak_prices`（单位：每 1M token 的美元），`cap` 是该模型的月额度上限；填了才会算「占限额」。
- 反代后面要关掉响应缓冲，否则流式会被攒成一坨再吐。
- 面板挂公网默认单步验证（`config.dashboard_auth.hosts`）。
- `responses`-`messages` 的互转是**可选且默认关**的（面板 key 页勾选），会丢保真度——见上面「协议互转」。
- **面板的「本机直连」是按 TCP 对端地址判定的**（loopback / 容器默认网关），不看 `Host` 之类可伪造的请求头；没配 `owner` 的旧 key 默认**只能跑 `/v1`、进不了面板**，本机脚本要用面板 API 才设 `GATEWAY_LEGACY_KEY_ADMIN=1`。
- 请求体有硬上限（`GATEWAY_MAX_BODY_MB`，默认 64MB）；面板接口额外拒绝 `Transfer-Encoding: chunked` 的请求（411）。带 base64 图片的客户端按需调大。
- 面板密码存 scrypt（`scrypt$N$r$p$salt$hash`），旧的 `sha256(salt+pw)` 记录仍能登录、**成功后自动升级**；据此改密/改角色会把该账号的旧会话踢掉。
- 统计里的 499 = 客户端断开、504 = 上游长时间无数据（看门狗）、502 = 上游失败或流传到一半被截断（带 `error` 字段）。

## 目录

```
server.mjs              网关核心（路由、三协议翻译、统计、面板 API）
dashboard.html          面板（单页、无构建，改完刷新即可）
config.example.json     配置模板
.env.example            上游 key 模板
CHANGELOG.md            更新日志
docker-compose.yml
scripts/smoke_test.py   冒烟脚本
scripts/interop_test/   协议互转矩阵回归（假上游 + 第二实例）
```

MIT
