# Agent Note: pi-ai 提供方请求上的 Harness 会话标头

Status: implemented

[English](2026-09-16-pi-ai-harness-session-header.md) | 中文

## Problem

循环会为每个模型请求盖上 `GenerateOptions.sessionId`。`dsh-llm-deepseek` 把该 id 映射为 `x-deepseek-harness-session-id`，而 `llm-pi-ai` 只把它传入 pi-ai 自身的流选项，在那里它用于选择提示缓存键，并在模型的 compat 启用时用于提供方的会话亲和标头。已安装目录中没有任何条目启用该开关，本包的 `compat` schema 也不暴露它，因此 pi-ai 路由完全不携带会话身份。

按会话路由或缓存的端点可能要求这一身份。OpenCode Go 对会话标头无法识别的请求返回 HTTP 400 `MissingSessionID`，使所有 OpenCode Go 模型都无法经该适配器使用。提供方无关接口本就预见了这种映射：`GenerateOptions.sessionId` 声明适配器可以将其作为模型不可见的传输元数据发送。

## Decision

`llm-pi-ai` 在每个携带会话的提供方请求上发送值为该请求确切 Session id 的 `x-deepseek-harness-session-id`，并在不携带会话的请求上省略该标头。使用 Harness 自身的标头名而非某个提供方专属的名称，是这两个适配器共享的协议事实：OpenCode Go 识别 Harness 的原生会话标头，因此同一个名称同时服务 `deepseek-official` 端点和每条 pi-ai 路由。

该标头加入适配器持有的名称集合。路由配置的 `headers` 仍会到达请求，而命名该会话标头的部署条目会被丢弃，不能顶替 Session id。

## Alternatives considered

**使用提供方专属名称，例如 `x-opencode-session`。** 端点确实接受它，但通用的多提供方适配器无法在不硬编码某家提供方词汇、或不增加一个指定标头名的路由字段的前提下选定它——而该字段会让每条 OpenCode Go 路由在部署方设置它之前都不可用。Harness 自身的名称已被端点识别，无需逐路由决策。

**为受影响的路由启用 pi-ai 的会话亲和标头。** pi-ai 仅在模型的 compat 设置 `sendSessionAffinityHeaders` 时才发送 `x-session-id` 与 `session_id`，而已安装的 OpenCode 目录条目都没有该设置；同一分支总会附加 `x-client-request-id` 与 `x-session-affinity`，而端点单独收到这两者时会拒绝。要到达唯一可接受的写法，需要在本包增加 `compat` schema 字段并在上游目录中开启该开关，而端点接受该标头恰恰只是因为前一种写法。

**增加一个命名会话标头的路由字段，默认不设置。** 这会让无需该信息的提供方对 Harness 一无所知，但这是一个只有一个已知消费者的部署侧可变选择，而且它会让 OpenCode Go 路由在用户配置该字段之前一直不可用。

## Consequences

每条已配置的 pi-ai 路由现在都会收到不透明的 Session id，即此前只到达 DeepSeek 官方端点的按会话标识符。无需该信息的提供方会忽略该标头；第三方提供方现在会得知这些请求属于不同会话。部署方不能再通过路由的 `headers` 设置该标头，因为 Harness 的值会取代它。

[协议扩展参考](../../../../docs/deepseek-llm-api-wire-extensions.zh.md) 继续持有该标头在 `deepseek-official` 请求中的取值与出现条件契约。其中记录了 `llm-pi-ai` 发送该标头、而不发送其定义的其他任何扩展，并将 pi-ai 的行为交由[包 README](../../../../packages/llm/llm-pi-ai/README.zh.md) 持有。

## Testing

`packages/llm/llm-pi-ai/tests/adapter.spec.ts` 覆盖了携带会话的请求、无法顶替 Session id 的同名部署 `headers` 条目，以及不发送会话标头的直接请求。
