# Agent Note: Per-conversation session header for provider routes

Status: implemented

[English](2026-09-09-provider-session-header.md) | 中文

[强制应用归属标头](../architecture/2026-06-21-mandatory-app-attribution-headers.zh.md)决策仍然拥有静态 `User-Agent` identity，并继续把会话 identity 排除在外。本决策新增一个独立的、按路由配置的请求标头，既不改变那些字段，也不改变那条规则，因此没有现存的 Agent Note 被取代。

## Problem

按会话路由或缓存的网关，只有在请求携带稳定会话 id 时才能这么做。OpenCode 的 Go 网关要求每个推理请求都带 `x-opencode-session`，否则返回 `400 MissingSessionID`；自 2026-09-05 起对所有客户端生效。`dsh-llm-deepseek` 会为其自有提供方发送 `x-deepseek-harness-session-id`，但 pi-ai 适配器——它服务每一个 OpenAI 兼容与 Anthropic 兼容路由，包括该网关——只发送静态归属标头。因此 `GenerateOptions.sessionId` 只到达 pi-ai 的 `sessionId` 选项（仅用于 pi-ai 自有标头格式内的提示词缓存亲和），从未以网关认识的标头形式到达线上请求。

pi-ai 的 `sendSessionAffinityHeaders` 兼容开关无法在这里补上这个缺口：该字段归目录所有，且被本包的漂移门禁挡在配置之外，默认关闭，其格式为 `session_id`、`x-client-request-id`、`x-session-affinity` 与 `x-session-id`——没有一个是该网关点名的标头。

## Decision

`PiAiProviderProfile.sessionHeader` 指定携带会话 id 的请求标头，`PiAiAdapter` 会把它加盖在每个 `GenerateOptions.sessionId` 存在的请求上。服务 OpenCode Go 网关的路由设置 `sessionHeader: x-opencode-session`：

```yaml
providers:
  opencode-go:
    apiKeyEnv: OPENCODE_API_KEY
    sessionHeader: x-opencode-session
```

取值即 harness 会话 id 原值，因此在同一会话的各轮次、恢复、压缩与重试之间保持稳定，且每个会话互不相同。点名会话的辅助调用——会话标题与压缩摘要——携带同一个值。配置的标头会替换同名的 `headers` 条目，因为单一固定值无法承担按会话 id 的职责；harness 署名标头在重名时仍然胜出，而空标头名或 Fetch 无法表示的标头名会在 profile 写入处被拒绝。未点名会话的请求——模型发现，或无密钥调用——不会发送该标头。

## Alternatives considered

**在 `headers` 里写固定值。** 这是最初的临时方案：`headers: { x-opencode-session: dsh-harness-opencode-session }`。否决其作为正式答案：单一固定值会把所有会话塌缩进同一个亲和桶——切换会话时网关仍指向同一副本，只是前缀不同，于是提示词缓存落空、成本上升。网关要的是按会话的 id。

**按路由键或端点主机名识别该网关并自动发送标头。** 否决：标头名是关于网关约定的一个事实，而不是提供方 identity 的属性。把路由命名为 `opencodego` 并指向该网关自身 baseURL 的部署需要一条名称或主机匹配规则才能被识别，而下一个有同样要求的网关又要再加一条规则。规范提供方细节属于 pi-ai，位于本适配器上游；在上游落地之前，一个显式的 profile 字段即可覆盖任意网关，无需主机白名单。

**在所有 pi-ai 路由上发送 `x-deepseek-harness-session-id`，与 `dsh-llm-deepseek` 对齐。** 否决：这会把 harness 命名的标头发往每一个第三方提供方，并依赖网关认识一个不在其文档约定内的名称。`dsh-llm-deepseek` 只把它发往定义该标头的端点。

**打开 pi-ai 的 `sendSessionAffinityHeaders` 兼容开关。** 否决，理由见 Problem：该字段被挡在配置之外、默认关闭，且没有任何亲和格式会发出 `x-opencode-session`。

## Consequences

位于会话亲和网关上的路由无需按会话的临时方案即可工作，其提示词缓存按会话保持温热，而不会塌缩进同一个桶。代价是每条这类路由多一项配置事实：需要它的部署必须设置它；决定网关是否看到 id 的是这个字段而非适配器，因此省略它的路由仍会像以前一样失败——对于约定未被满足的网关，这正是预期的响亮行为。

该 id 即 harness 会话 id，属于模型不可见的传输元数据：它没有任何部分进入提示词、会话日志或模型可见请求。未点名会话的请求仍然什么都不发，因此要求该标头的网关会拒绝发现探测与无密钥调用；本字段无法为一个没有会话的请求凭空造出 id。

## Testing

`packages/llm/llm-pi-ai/tests/adapter.spec.ts` 驱动真实本地 HTTP 服务器，断言带会话的请求以会话 id 收到该标头，且未点名会话的请求保留部署配置的同名静态条目。同一文件还断言解析会拒绝空标头名以及 Fetch 无法表示的标头名。
