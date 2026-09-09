# Agent Note: Per-conversation session header for provider routes

Status: implemented

English | [中文](2026-09-09-provider-session-header.zh.md)

The [mandatory app attribution headers](../architecture/2026-06-21-mandatory-app-attribution-headers.md) decision still owns the static `User-Agent` identity and keeps session identity out of it. This decision adds a separate, per-route request header and changes neither those fields nor that rule, so no active Agent Note is superseded.

## Problem

A gateway that routes or caches by conversation can only do so when a request carries a stable conversation id. OpenCode's Go gateway requires `x-opencode-session` on every inference request and answers `400 MissingSessionID` without it, for all clients from 2026-09-05. `dsh-llm-deepseek` sends `x-deepseek-harness-session-id` for its own provider, but the pi-ai adapter — which serves every OpenAI-compatible and Anthropic-compatible route, including that gateway — sent only the static attribution header. `GenerateOptions.sessionId` therefore reached pi-ai's `sessionId` option, which only feeds prompt-cache affinity inside pi-ai's own header formats, and never reached the wire as a header the gateway recognizes.

pi-ai's `sendSessionAffinityHeaders` compat switch cannot close that gap here: it is catalog-owned and withheld from configuration by this package's drift gates, defaults to false, and its formats are `session_id`, `x-client-request-id`, `x-session-affinity`, and `x-session-id` — none of which is the header the gateway names.

## Decision

`PiAiProviderProfile.sessionHeader` names the request header that carries the conversation id, and `PiAiAdapter` stamps it on every request whose `GenerateOptions.sessionId` is present. A route serving OpenCode's Go gateway sets `sessionHeader: x-opencode-session`:

```yaml
providers:
  opencode-go:
    apiKeyEnv: OPENCODE_API_KEY
    sessionHeader: x-opencode-session
```

The value is the harness session id verbatim, so it is stable across one conversation's turns, resumes, compaction, and retries, and distinct per conversation. Auxiliary calls that name the conversation — session titles and compaction summaries — carry the same value. The configured header replaces a same-named `headers` entry, because one fixed value cannot do a per-conversation id's job; attribution still wins a reserved name, and resolution refuses an empty or Fetch-unrepresentable header name where the profile is written. A request that names no session — model discovery, or a keyless call — sends nothing under it.

## Alternatives considered

**A static value in `headers`.** This was the immediate workaround: `headers: { x-opencode-session: dsh-harness-opencode-session }`. Rejected as the shipped answer because one fixed value collapses every conversation into one affinity bucket: switching sessions keeps pointing the gateway at the same replica with a different prefix, so prompt caches miss and cost rises. A per-conversation id is what the gateway asks for.

**Detect the gateway from the route key or the endpoint host and send the header automatically.** Rejected because the header name is a fact about the gateway's contract, not about a provider identity: a deployment that names its route `opencodego` and points it at the gateway's own baseURL would need a name or host match to be recognized, and the next gateway with the same requirement would need another rule. Normalizing a provider's particulars belongs in pi-ai, upstream of this adapter; until that ships, one explicit profile field covers any gateway without a host allowlist.

**Send `x-deepseek-harness-session-id` on every pi-ai route, mirroring `dsh-llm-deepseek`.** Rejected because it puts a harness-named header on requests to every third-party provider and relies on a gateway recognizing a name outside its documented contract. `dsh-llm-deepseek` sends it to the endpoint that defines it.

**Enable pi-ai's `sendSessionAffinityHeaders` compat switch.** Rejected for the reasons in the Problem: the field is withheld from configuration, defaults to false, and no affinity format emits `x-opencode-session`.

## Consequences

A route on a conversation-affinity gateway works without a per-session workaround, and its prompt cache stays warm per conversation instead of collapsing into one bucket. The cost is one more configuration fact per such route: a deployment that needs it must set it, and this field — not the adapter — decides whether the gateway sees an id, so a route that omits it keeps failing exactly as before, which is the intended loud behavior for a gateway whose contract is not met.

The id is the harness session id, so it is model-hidden transport metadata: nothing about it enters the prompt, the session log, or the model-visible request. Requests that name no session still send nothing, so a gateway requiring the header refuses discovery probes and keyless calls; this field cannot invent an id for a request that has no conversation.

## Testing

`packages/llm/llm-pi-ai/tests/adapter.spec.ts` drives a real local HTTP server and asserts the header arrives with the conversation id on a session-bearing request and that a request naming no session keeps the deployment's static same-named entry. The same file asserts that resolution refuses an empty header name and a name Fetch cannot represent.
