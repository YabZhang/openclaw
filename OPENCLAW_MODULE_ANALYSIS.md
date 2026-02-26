# OpenClaw Module Analysis

This document maps the OpenClaw architecture to its source code structure. Use this to navigate the codebase and understand where specific responsibilities live.

## 1. Core Runtime: `src/gateway/`

The **Gateway** is the application server. It stays running, manages connections, and orchestrates everything else.

| Module | Responsibility | Key Files |
| :--- | :--- | :--- |
| **Server** | Entry point, startup logic, and dependency injection. | `server.impl.ts`, `server.ts` |
| **Runtime State** | In-memory state (clients, active runs, shared resources). | `server-runtime-state.ts` |
| **WebSocket** | Handling client connections (CLI, WebChat, macOS app). | `server-ws-runtime.ts` |
| **Methods** | RPC methods exposed to clients (`sessions_list`, `chat_send`). | `server-methods.ts`, `server-methods/*` |
| **Channels** | Managing the lifecycle of messaging adapters. | `server-channels.ts`, `channel-health-monitor.ts` |

**Concept Mapping**:
*   `GatewayServer` (Architecture) -> `src/gateway/server.impl.ts`
*   `GatewayRuntimeState` (Architecture) -> `src/gateway/server-runtime-state.ts`

## 2. Agent Engine: `src/agents/`

The **Agent** logic handles the "thinking" and execution loops. It is stateless between runs (state is persisted to disk).

| Module | Responsibility | Key Files |
| :--- | :--- | :--- |
| **Runner** | The core loop: Prompt -> LLM -> Tool -> Result -> Repeat. | `pi-embedded-runner.ts`, `pi-embedded-runner/run/attempt.ts` |
| **Context** | Managing token limits, compaction, and history truncation. | `context-window-guard.ts`, `pi-embedded-runner/compact.ts` |
| **Tools** | implementations of tools available to the agent. | `tools/`, `bash-tools.ts`, `openclaw-tools.ts` |
| **Sandbox** | Docker container management for secure execution. | `sandbox/docker.ts`, `sandbox/config.ts` |
| **Auth** | Model API key rotation and cooldown management. | `auth-profiles.ts`, `model-auth.ts` |

**Concept Mapping**:
*   `PiEmbeddedRunner` (Architecture) -> `src/agents/pi-embedded-runner/`
*   `Agent Session` (Architecture) -> Managed via `@mariozechner/pi-coding-agent` (imported in `attempt.ts`)

## 3. Communication Layer: `src/channels/`

**Channels** act as bridges between external messaging platforms and the Gateway.

| Module | Responsibility | Key Files |
| :--- | :--- | :--- |
| **Telegram** | Polling/Webhook bot for Telegram. | `telegram/bot.ts`, `telegram/bot-handlers.ts` |
| **Routing** | Resolving `chat_id` to `session_key`. | `routing/resolve-route.ts` |
| **Plugins** | Interfaces for channel plugins. | `channels/plugins/` |

**Note**: Other channels (WhatsApp, Discord, Slack) typically live in their own folders (e.g., `src/discord`, `src/slack`) or as external plugins.

## 4. Infrastructure & Utilities: `src/infra/`

Shared utilities that underpin the entire system.

| Module | Responsibility | Key Files |
| :--- | :--- | :--- |
| **Execution** | Safe command execution, approval workflows, shell path resolution. | `exec-approvals.ts`, `exec-safe-bin-policy.ts` |
| **Outbound** | Reliable message delivery queue. | `outbound/delivery-queue.ts` |
| **Persistence** | File locking, atomic writes, JSON handling. | `fs-safe.ts`, `json-file.ts` |
| **Network** | Port checking, Bonjour discovery, Tailscale integration. | `ports.ts`, `bonjour.ts`, `tailscale.ts` |
| **Updates** | Self-update checks and version management. | `update-check.ts` |

## 5. Configuration: `src/config/`

How the system loads and validates user settings.

| Module | Responsibility | Key Files |
| :--- | :--- | :--- |
| **Loading** | Reading `~/.openclaw/openclaw.json` and merging defaults/env. | `config.ts` |
| **Schema** | TypeBox definitions for strict config validation. | `config-schema.ts` |
| **Hot Reload** | Watching config files for changes. | `../gateway/config-reload.ts` (relies on config module) |

## 6. Plugins: `src/plugins/`

The extensibility layer.

| Module | Responsibility | Key Files |
| :--- | :--- | :--- |
| **Registry** | Loading, validating, and registering plugins. | `registry.ts` |
| **Hooks** | Defining and running lifecycle hooks (`before_prompt_build`). | `hook-runner-global.ts` |
| **Loader** | Dynamic import of plugin modules. | `loader.ts` |

## Navigation Tips

*   **Where does the app start?** `src/entry.ts` -> `src/cli/run-main.ts` -> `src/gateway/server.impl.ts`.
*   **Where is the LLM called?** `src/agents/pi-embedded-runner/run/attempt.ts` -> `activeSession.prompt()`.
*   **Where are commands blocked?** `src/infra/exec-approvals.ts` and `src/agents/bash-tools.exec.ts`.
*   **Where do messages come in?** Search for `processInboundMessage` in `src/channels` or specific channel folders (e.g., `src/telegram/bot.ts`).
