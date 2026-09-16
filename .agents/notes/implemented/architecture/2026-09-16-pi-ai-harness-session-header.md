# Agent Note: Harness session header on pi-ai provider requests

Status: implemented

English | [中文](2026-09-16-pi-ai-harness-session-header.zh.md)

## Problem

The loop stamps every model request with `GenerateOptions.sessionId`. `dsh-llm-deepseek` maps that id to `x-deepseek-harness-session-id`, but `llm-pi-ai` passed it only into pi-ai's own stream options, where it selects a prompt-cache key and, for a model whose compat enables them, the provider's session-affinity headers. None of the installed catalog entries enables them and this package's `compat` schema does not expose the switch, so a pi-ai route carried no session identity at all.

An endpoint that routes or caches by conversation can require one. OpenCode Go answers a request whose session header it does not recognize with HTTP 400 `MissingSessionID`, which made every OpenCode Go model unusable through this adapter. The provider-neutral interface already anticipates the mapping: `GenerateOptions.sessionId` documents that an adapter may send it as model-hidden transport metadata.

## Decision

`llm-pi-ai` sends `x-deepseek-harness-session-id` holding the request's exact Session id on every provider request that carries one, and omits the header from a request that does not. The Harness's own header name, rather than a provider-specific one, is the wire fact both adapters share: OpenCode Go recognizes the Harness's native session header, so one name serves the `deepseek-official` endpoint and every pi-ai route.

The header joins the set of names the adapter owns. A route's configured `headers` still reach the request, and a deployment entry naming the session header is dropped rather than displacing the Session id.

## Alternatives considered

**A provider-specific name such as `x-opencode-session`.** The endpoint does accept it, but a generic multi-provider adapter cannot choose one provider's vocabulary without either hardcoding it or adding a route field that names the header — and that field would leave every OpenCode Go route broken until a deployment set it. The Harness's own name is recognized by the endpoint and needs no per-route decision.

**Enable pi-ai's session-affinity headers for the affected routes.** pi-ai sends `x-session-id` and `session_id` only when a model's compat sets `sendSessionAffinityHeaders`, which no installed OpenCode catalog entry does; the same branch always adds `x-client-request-id` and `x-session-affinity`, which the endpoint rejects as session headers on their own. Reaching the one acceptable spelling would require a `compat` schema addition here and the switch in the upstream catalog, for a header the endpoint accepts only because of that first spelling.

**A route field naming the session header, unset by default.** This would keep providers that have no use for the Harness unaware of it, but it is a deployment-varying choice with a single known consumer, and it ships the OpenCode Go route non-functional until a user configures it.

## Consequences

Every configured pi-ai route now receives the opaque Session id, a per-conversation identifier that previously reached only the official DeepSeek endpoint. A provider with no use for it ignores the header; a third-party provider now learns that requests belong to distinct conversations. A deployment can no longer set that header through a route's `headers`, because the Harness value replaces it.

The [wire extension reference](../../../../docs/deepseek-llm-api-wire-extensions.md) keeps owning the header's value and presence contract for `deepseek-official` requests. It records that `llm-pi-ai` sends this header and none of the other additions it defines, and it defers to the [package README](../../../../packages/llm/llm-pi-ai/README.md) for the pi-ai behavior.

## Testing

`packages/llm/llm-pi-ai/tests/adapter.spec.ts` covers a request carrying a Session, a deployment `headers` entry of the same name that cannot displace the Session id, and a direct request, which sends no session header.
