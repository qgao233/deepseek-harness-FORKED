# Agent Note: Per-server MCP protocol-era negotiation mode

Status: implemented

English | [中文](2026-10-08-mcp-per-server-version-negotiation.zh.md)

## Problem

The official client 2.0.0 sends a modern-era `server/discover` probe before the `initialize` handshake when the bridge constructs its `Client` with `versionNegotiation: { mode: 'auto' }` (see the [archived SDK-negotiation Agent Note](../../archived/feature/2026-09-12-mcp-sdk-protocol-negotiation.md)). A legacy-era server that cannot handle the unknown probe method may answer with an HTTP 5xx instead of a JSON-RPC method-not-found error. The SDK deliberately classifies a probe 5xx as a broken server — a typed connect error with no legacy fallback — while 4xx, JSON-RPC errors, and stdio silence do fall back. Such a server is perfectly reachable through the plain handshake, yet the connection can never establish: the supervisor burns its reconnect budget and the server stays disconnected, with no remedy because the mode was hardcoded in the bridge. A real deployment showed this shape: Aliyun Bailian's Streamable HTTP WebSearch endpoint answers the probe with HTTP 500 while the plain handshake succeeds and negotiates protocol 2024-11-05.

## Decision

`dsh-mcp-client` exposes a per-server `versionNegotiation` config with a `mode` field that mirrors the SDK's `ClientOptions.versionNegotiation.mode`: `'auto'` (the default, unchanged behavior), `'legacy'` (plain `initialize` handshake, no probe), or `{ pin: '<revision>' }` (modern era at exactly that revision, no fallback). Both transports accept the field; stdio's disposable sibling-probe semantics are unchanged. The supervisor always passes an explicit mode, so the DSH default stays `'auto'` even though the SDK's own default is `'legacy'`. The schema validates a pin as an ISO-date revision; the SDK's modern-era boundary check remains the connect-time authority.

## Alternatives considered

**Hardcode the bridge default back to `legacy`.** This repairs affected servers but silently drops modern-era discovery for every server that supports `server/discover`, and it removes any route to a pinned modern revision.

**Classify a probe 5xx as legacy evidence in the bridge.** The SDK owns the classification. Overriding "5xx means broken" hides genuinely broken servers behind a handshake that papers over the symptom, and it forks maintained SDK behavior.

**Patch the pinned SDK classifier.** The upstream is still evolving; a local patch to negotiation semantics is a maintenance liability that a configuration field does not need.

**Retry the same URL in legacy mode after the first probe 5xx.** This doubles connect attempts against every broken server and turns what should be an explicit operator choice into hidden per-outcome branching inside the reconnect budget.

## Consequences

- A legacy-era server that rejects the probe has a configuration remedy: set `versionNegotiation: { mode: 'legacy' }` on that entry; unchanged entries keep probe-and-fallback behavior.
- Unreachable servers are now distinguishable: a probe 5xx names the negotiation failure in the log, and the operator chooses the era instead of the bridge guessing.
- A syntactically valid but non-modern pin still fails at connect with the SDK's typed error and consumes the reconnect budget; the schema date-shape check catches only obvious typos.
- Tests pin the schema defaults, mode validation, and the exact pass-through into the SDK `Client` options; the live-server probe-500-versus-legacy outcome was verified manually against the deployment above, not in CI.
