# 只读作用域（scope=quota）：给监控用的客户端 key

## 为什么需要

网关的虚拟 key 默认是「全功能」：能对话、能生成、也能读额度。但监控面板、桌面悬浮窗这类程序只需要读数字，不需要、也不应该具备消费能力。一旦这类程序的配置文件泄漏，全功能 key 会把上游额度一起送出去。

`scope` 字段把这件事显式化：同一个网关、同一套 key 表，只读 key 拿不到对话与生成路径。

## 配置

在 `config.json` 的 `keys` 数组里给条目加一个字段：

```json
{
  "name": "quota-reader",
  "key": "sk-gw-example-readonly",
  "scope": "quota",
  "session_id": "00000000-0000-0000-0000-000000000002"
}
```

- `scope` 省略或写成 `all`：与旧版行为完全一致，不做任何限制。
- `scope: "quota"`：进入只读模式，规则见下表。
- 该字段是 key 级配置，不影响其他 key，也不需要重启后重新签发。

## 只读模式下允许的请求

| 路径 | 说明 |
|---|---|
| `GET /v1/models` | 列模型，供客户端探活与下拉框 |
| `GET /v1/models/{id}` | 单模型查询 |
| `GET /api/upstream/quota` | 上游额度汇总（需该 key 有读取权限） |
| `GET /api/stats` | 网关自身用量统计 |

路径判定按前缀 `/v1` 与 `/v1beta` 进行，Gemini 原生端点（`/v1beta/models/{model}:generateContent`）同样被拦。

其余 `/v1` 与 `/v1beta` 前缀的路径（`chat/completions`、`responses`、`messages`、Gemini 原生 `generateContent` 等）一律返回：

```json
{
  "error": {
    "message": "read-only key: only model listing and quota queries are permitted",
    "type": "permission_error",
    "code": "read_only_key"
  }
}
```

HTTP 状态码 `403`。

## 限速

只读 key 的 `/v1` 请求限制为**每秒 1 次**（按 key 名计，进程内计数，重启清零；被拒绝的请求同样计数，避免刷 403）。超过时返回 `429`，并带 `Retry-After: 1`：

```json
{
  "error": {
    "message": "read-only key is limited to one request per second",
    "type": "rate_limit_error",
    "code": "rate_limited"
  }
}
```

额度查询接口不受这条限速影响，它自身有缓存；建议客户端轮询间隔不低于 60 秒。

## 使用示例

```bash
# 列模型
curl -s https://gateway.example.com/v1/models \
  -H "Authorization: Bearer sk-gw-example-readonly"

# 读额度
curl -s https://gateway.example.com/api/upstream/quota \
  -H "Authorization: Bearer sk-gw-example-readonly"

# 试图对话 → 403 read_only_key
curl -s -o /dev/null -w '%{http_code}\n' \
  -X POST https://gateway.example.com/v1/chat/completions \
  -H "Authorization: Bearer sk-gw-example-readonly" \
  -H "Content-Type: application/json" \
  -d '{"model":"example-model","messages":[{"role":"user","content":"hi"}]}'
```

## 安全边界

- 只读 key 不含任何上游凭据，网关在服务端替换为真实上游 key；key 本身泄漏的最坏后果是他人能看到模型清单与额度数字。
- 只读 key 不能修改配置：配置接口只接受面板登录或本机对端，虚拟 key 一律 401。
- 与模型白名单（`keys[].models`）可以叠加：先按作用域判定能不能用这条路径，再按白名单判定能用哪个模型。

## 验证清单

1. `GET /v1/models` 返回 200 与模型清单。
2. `POST /v1/chat/completions` 返回 403，`code=read_only_key`。
3. 连续两次 `GET /v1/models`，第二次返回 429，响应头含 `Retry-After: 1`。
4. 把 `scope` 改回 `all`（或删掉）并重载配置，对话恢复 200。
