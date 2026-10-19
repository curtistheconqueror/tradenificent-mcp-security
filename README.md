# TRADENIFICENT MCP Security Architecture

**Author:** Curtis ([@curtistheconqueror](https://github.com/curtistheconqueror))

TRADENIFICENT is a multi-agent futures-trading intelligence platform. AI agents (Claude, ChatGPT/Codex and Grok-based bots) research markets, monitor live data, and run strategy bots against prop-firm accounts. TRADENIFICENT exposes a **read-only Model Context Protocol (MCP) endpoint**, so other people's AI agents can plug in and pull market context and trade ideas.

Agents with real money behind them leave no room for sloppy security. This document explains how the MCP layer is designed to protect the people who connect to it, the platform itself, and the accounts behind it.

## Core principle
**No connected agent can ever cause an order to be placed.** TRADENIFICENT is an information service. The external MCP surface contains no order-capable tools, an automated test checks this on every build, and trade calls are framed as *ideas*, not instructions.

## Threat model
- **Assets:** live broker accounts; broker, market-data and LLM API keys; proprietary strategy logic; the host infrastructure; and connecting clients' own data.
- **Threats:** prompt injection through untrusted content (OWASP LLM01); malicious or hijacked MCP servers and packages (rug-pull updates); over-scoped or long-lived credentials; tool-name collisions; data exfiltration; and runaway or abusive clients (denial of service, denial of wallet).

## 1. Supply-chain integrity
- Every MCP server TRADENIFICENT runs is pinned to an immutable commit and SHA-256 digest. Mutable tags like `latest` are never used, and a running server's hash must match its approved value.
- Third-party connectors get forked and stripped to the minimum read-only tool set before use. For example, a community TradingView connector was cut down to 21 read-only tools, verified against a 64-test suite offline by a dedicated QA agent, and shipped only as a hash-verified bundle with before/after snapshots and a one-step rollback.
- An internal allowlist of approved server digests is enforced, and CI rejects unsigned or unpinned artifacts.
- Tools are namespaced by server, so no two servers can expose a tool with the same name.

## 2. Least privilege
- **Per-client credentials:** every connecting agent gets its own short-lived, rotatable API key, scoped only to read-only tools and kept separate from administrative credentials. Revoking one client never affects another.
- **Agent and human roles are separate:** each internal agent has one scoped job (monitor, tester, researcher or trader). Monitors are read-only, and no agent uses an administrator's identity.
- **Policy beats annotations:** the gateway enforces its own allow/deny policy per tool and doesn't rely on a tool's self-declared `readOnlyHint`. Write-style requests are blocked at the policy layer.
- **Narrow internal permissions:** internal coding agents run behind a permission classifier that blocks real-world transactions by default. Exceptions are granted as exact-prefix rules (one HTTP method, one endpoint, one agent ID), never as blanket bypasses.

## 3. Untrusted content and human-in-the-loop
- All external content (web pages, tool results and other agents' messages) is treated as data, never as instructions. Instructions embedded in that content get surfaced for human review, not executed.
- Responses are sanitized before they reach a model: code blocks are neutralized, command-injection characters are escaped, and markdown that could exfiltrate data through remote image or link loads is stripped.
- Any irreversible internal action, such as changing a live bot's configuration, sending an outbound message or deleting data, needs explicit human confirmation with a preview of the exact action.
- New servers and tools start in manual-review mode and earn auto-run status only after review.
- Hard financial risk limits rely on the brokers' own tested, server-side controls (daily loss and maximum loss limits) as the final backstop.

## 4. Isolation and sandboxing
- The MCP server runs as a non-root user in a container with privilege escalation disabled.
- An egress allowlist lets the server reach only approved market-data APIs. All other outbound traffic is dropped.
- CPU and memory quotas cap the damage from a compromised or abusive workload.
- High-risk connectors run in dedicated sessions with shell, web-fetch and web-search tools denied, and they're never loaded into the global agent configuration.

## 5. Centralized gateway
- All external MCP traffic passes through one gateway over TLS, authenticated per client.
- **Rate limiting:** per-key quotas and burst throttling stop runaway scripts and abuse.
- **Inline guardrails:** pattern-based prompt-injection filters and secret redaction run on every request and response.
- **Full audit logging:** every call is logged as structured JSON with a request ID, client ID, tool ID, arguments, latency and response size, and logs are written to append-only storage.
- **Secret hygiene:** no agent ever prints or echoes API keys, tokens or credentials, and keys never appear in chat, logs or responses.

## 6. What this means for people who connect
- Your agent can only *read*. There's no tool that can trade, write or reach outside the approved data surface.
- Your key is yours alone and can be revoked instantly without affecting anyone else.
- Every call is attributable and auditable.
- Content your agent receives has been sanitized against injection and exfiltration tricks.

## 7. How success is measured
- 100% of external MCP calls attributable to a client ID and tool ID.
- 100% of running MCP servers pinned to a verified digest.
- Zero order-capable tools exposed externally, verified automatically.
- Time to revoke a compromised client key measured in minutes, and critical-issue remediation in under 4 hours.
