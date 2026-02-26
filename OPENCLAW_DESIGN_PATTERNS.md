# OpenClaw: Lessons in Agent Design

This document synthesizes key architectural patterns and design decisions from the OpenClaw codebase. It answers "how it works so well" by highlighting the specific engineering choices that contribute to its reliability, safety, and capability.

## 1. Reliability Engineering: "Graceful Degradation"

OpenClaw assumes that everything external (LLMs, networks, APIs) will eventually fail. The system is built to handle these failures transparently.

### 1.1. The Failover Pipeline
Instead of a simple try/catch, OpenClaw implements a sophisticated `FailoverError` system (`src/agents/failover-error.ts`).
*   **Classification**: Errors are classified into types like `rate_limit`, `billing`, `auth`, `timeout`, and `context_overflow`.
*   **Routing**: The `runWithModelFallback` function (`src/agents/model-fallback.ts`) iterates through a candidate list.
    *   *Primary Model* -> *Configured Fallback* -> *Next Provider*.
*   **Cooldowns**: If an auth profile fails, it is marked with a cooldown in `AuthProfileStore`. The system automatically skips "cold" profiles to prevent wasted retries.

### 1.2. The Delivery Queue
Outbound messages are not "fire and forget". They use a persistent `DeliveryQueue` (`src/infra/outbound/delivery-queue.ts`).
*   **Persistence**: Messages are written to disk (`delivery-queue/*.json`) *before* transmission attempts.
*   **Backoff**: Failures trigger exponential backoff (5s, 25s, 2m, 10m).
*   **Recovery**: On startup, the Gateway scans the queue and resumes pending deliveries, ensuring no messages are lost during a crash or restart.

## 2. Context Economy: Managing the Finite Resource

Tokens are treated as a scarce resource. The system aggressively optimizes context usage to maintain performance and reduce costs over long sessions.

### 2.1. Aggressive Tool Result Truncation
Large outputs (like `cat huge_file.log`) are the enemy of context windows.
*   **The 30% Rule**: A single tool result is capped at ~30% of the model's context window (`src/agents/pi-embedded-runner/tool-result-truncation.ts`).
*   **Truncation Logic**: If a result exceeds the limit, it is truncated with a suffix warning: *"Content truncated... use offset/limit to read smaller chunks."*
*   **Session Repair**: The system can retroactively truncate *past* tool results in the session history if a new model with a smaller context window is selected.

### 2.2. Intelligent Compaction
When the context limit is approached, the system doesn't just drop messages.
*   **Compaction Loop**: The `compactEmbeddedPiSessionDirect` function (`src/agents/pi-embedded-runner/compact.ts`) triggers a recursive summarization process.
*   **Summarization**: Old turns are summarized into a narrative block, preserving key facts while reclaiming tokens.
*   **Safety Timeout**: Compaction runs with a strict timeout to prevent infinite loops from stalling the agent.

## 3. Safety by Design: "Defense in Depth"

OpenClaw allows agents to run shell commands, which is inherently dangerous. Security is layered.

### 3.1. Docker Sandboxing
The default execution environment for non-main sessions is a Docker container (`src/agents/sandbox/docker.ts`).
*   **Hardening**: Containers are created with `--cap-drop all`, `--security-opt no-new-privileges`, and strict seccomp profiles.
*   **Mounts**: Only specific workspace directories are mounted. `validateSandboxSecurity` prevents dangerous binds (like `/`) unless explicitly opted-in via "dangerouslyAllow..." flags.
*   **Isolation**: Network access can be restricted, and PIDs are limited.

### 3.2. Command Pre-flight
Before executing a command, the `execTool` performs heuristic checks (`src/agents/bash-tools.exec.ts`).
*   **Shell Bleed Detection**: Scans Python/Node scripts for accidental shell variable injection (e.g., using `$VAR` instead of `os.environ`).
*   **Elevation Gates**: `sudo` access requires explicit "elevated" permission configuration.

### 3.3. Approval Workflows
Critical actions can trigger an "Approval Request" flow. The Gateway holds the execution, sends a request to the user (via the control UI or chat), and waits for an explicit "Approve" signal before proceeding.

## 4. Architecture: The "Gateway" Pattern

The separation of concerns between the **Gateway** (Control Plane) and **Agents** (Execution Plane) is key to the system's robustness.

*   **Gateway as Sovereign**: The Gateway manages the WebSocket server, plugin registry, and channel connections. It survives agent crashes.
*   **Agents as Ephemeral**: Agents are spun up on demand for a session and can be terminated without bringing down the whole system.
*   **State Management**: `GatewayRuntimeState` acts as the in-memory source of truth, managing concurrent connections and routing logic efficiently.

## 5. Extensibility: The Plugin System

The plugin architecture allows the core to remain lean while enabling vast capability.
*   **Hooks**: The lifecycle hooks (`before_prompt_build`, `llm_input`, `agent_end`) allow plugins to intercept and modify every stage of the agent loop.
*   **Registry**: A central `PluginRegistry` (`src/plugins/registry.ts`) manages dependencies and ensures plugins play nicely together.

## Summary: Why it works well

OpenClaw works well because it **anticipates failure** and **constraints**.
1.  It assumes models will fail (and has fallbacks).
2.  It assumes context is limited (and has compaction/truncation).
3.  It assumes code execution is dangerous (and has sandboxing/approvals).
4.  It assumes networks are flaky (and has delivery queues).

It turns these constraints into architectural features rather than afterthoughts.
