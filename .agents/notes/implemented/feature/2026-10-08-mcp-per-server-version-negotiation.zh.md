# Agent Note: MCP 协议时代协商按服务器配置

Status: implemented

[English](2026-10-08-mcp-per-server-version-negotiation.md) | 中文

## Problem

当桥接器以 `versionNegotiation: { mode: 'auto' }` 构造 `Client` 时，官方 client 2.0.0 会在 `initialize` 握手之前先发送现代时代的 `server/discover` 探测（见[已归档的 SDK 协商 Agent Note](../../archived/feature/2026-09-12-mcp-sdk-protocol-negotiation.md)）。无法处理这个未知探测方法的旧时代服务器可能返回 HTTP 5xx，而不是 JSON-RPC 的 method-not-found 错误。SDK 有意把探测 5xx 分类为服务器损坏——带类型的连接错误、不回退旧版——而 4xx、JSON-RPC 错误与 stdio 静默都会回退。这样的服务器本可通过普通握手正常连接，但连接永远无法建立：监督器耗尽重连预算后，服务器停留在 disconnected 状态，且因为模式硬编码在桥接器里而没有任何补救手段。一个真实部署正是这种形态：阿里云百炼的 Streamable HTTP WebSearch 端点对探测返回 HTTP 500，而普通握手成功并协商到协议 2024-11-05。

## Decision

`dsh-mcp-client` 暴露按服务器的 `versionNegotiation` 配置，其 `mode` 字段镜像 SDK 的 `ClientOptions.versionNegotiation.mode`：`'auto'`（默认，行为不变）、`'legacy'`（仅普通 `initialize` 握手，不探测）、或 `{ pin: '<revision>' }`（恰好锁定该现代修订版，不回退）。两种传输都接受该字段；stdio 的可释放兄弟探测语义不变。监督器始终传入显式模式，因此即使 SDK 自身默认是 `'legacy'`，DSH 的默认仍是 `'auto'`。schema 校验 pin 为 ISO 日期修订版；现代时代边界检查仍由 SDK 在连接时裁决。

## Alternatives considered

**把桥接器默认值硬编码回 `legacy`。** 这能修复受影响的服务器，却为所有支持 `server/discover` 的服务器悄悄丢掉现代时代发现能力，也堵死了锁定现代修订版的路径。

**在桥接器内把探测 5xx 分类为旧版证据。** 分类由 SDK 负责。覆盖「5xx 意味着损坏」会让真正损坏的服务器被一次掩盖症状的握手藏起来，并且分叉了已维护的 SDK 行为。

**给固定版本的 SDK 分类器打补丁。** 上游仍在演进；对协商语义打本地补丁是一份配置字段所不需要的维护负担。

**首次探测 5xx 后自动以 legacy 模式重试同一 URL。** 这会对每个损坏服务器成倍消耗连接尝试，还把本应明确的运维选择变成重连预算里隐藏的按结果分支。

## Consequences

- 拒绝探测的旧时代服务器有了配置补救：在该条目上设置 `versionNegotiation: { mode: 'legacy' }`；未改动的条目保持探测并回退的行为。
- 不可达服务器如今可以区分：探测 5xx 会在日志里点名协商失败，由运维选择协议时代，而不是桥接器猜测。
- 语法合法但非现代时代的 pin 仍会在连接时以 SDK 的带类型错误失败并消耗重连预算；schema 的日期形状检查只能拦住明显笔误。
- 测试固定了 schema 默认值、模式校验，以及传入 SDK `Client` 选项的确切透传；真实服务器上「探测 500 对比 legacy 成功」的结果是对上述部署的人工验证，不在 CI 中。
