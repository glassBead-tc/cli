# Extending the Smithery MCP CLI to Full MCP Capability Coverage

This document evaluates the current foundation of the CLI, maps it to the Model Context Protocol (MCP) capability set, and proposes a concrete, file-by-file implementation plan to support all currently available MCP capabilities: Resources, Tools, Prompts, Discovery, Roots, Sampling, and Elicitation.

## Executive summary

- This CLI already has a solid foundation for a full-featured MCP client toolchain:
  - Robust command surface for end users: `install`, `uninstall`, `run`, `inspect`, `list`, `dev`, `build`, `playground` in `src/index.ts`.
  - Dual-transport support for servers: stdio and Streamable HTTP via `src/commands/run/stdio-runner.ts` and `src/commands/run/streamable-http-runner.ts`.
  - Development ergonomics: hot-reload dev server and bundling via `src/commands/dev.ts` and `src/lib/build.ts`, with bootstraps in `src/runtime/*`.
  - Uses `@modelcontextprotocol/sdk` and already wires capabilities for resources/tools/prompts listing in `src/commands/inspect.ts`.
  - Registry integration and environment bootstrapping (UV/Bun detection, API key handling).
- To reach “full MCP capability coverage,” we primarily need to:
  1. Expand the interactive inspector to exercise and demonstrate all capabilities (Tools execution, Resources, Prompts, Discovery/subscriptions, Roots, Sampling, Elicitation).
  2. Expose optional client capabilities (sampling, elicitation) with pluggable backends.
  3. Improve runner ergonomics for cancellation, progress, and discovery notifications (pass-through is mostly complete already).
  4. Add targeted docs/tests and developer guidance.

## Current capability coverage (at a glance)

- Resources: list + read are supported in `inspect`.
- Tools: list is supported in `inspect`; execution is not currently implemented there.
- Prompts: list + get are supported in `inspect`.
- Discovery (updates/subscriptions): not implemented; only static lists.
- Roots: not surfaced in the CLI; not in `inspect`.
- Sampling: not implemented (CLI does not expose a sampling backend to servers).
- Elicitation: not implemented (no interactive response path from server to user via CLI).

The `run` command already acts as a transparent JSON‑RPC proxy between the LLM and the server over stdio or Streamable HTTP. That means many advanced features (e.g., sampling, discovery) can flow through if the upstream LLM client supports them. However, to claim full coverage within this CLI, we should add first-class interactive support for these features in `inspect` and optional client capabilities where relevant.

## Guiding principles

- Maintain the thin, transparent nature of `run` so it remains an LLM‑friendly bridge.
- Consolidate “demo and debug” of all capabilities inside `inspect`.
- Keep sampling and elicitation optional and pluggable to avoid hard vendor lock‑in.
- Prefer minimal surface changes to existing commands; add flags where needed.

## Roadmap by capability

### 1) Tools

- Gaps:
  - `inspect` lists tool schemas but does not execute tools.
  - No streaming/progress display during tool execution.
- Plan:
  - Add interactive tool execution in `src/commands/inspect.ts`:
    - Prompt for tool input using its JSON Schema, including nested objects and arrays.
    - Execute via `client.callTool({ name, arguments })`.
    - Display rich results (text, JSON content) and capture any `progress` or `logging` notifications.
  - Add a `--json` flag to dump raw results for automation.
  - Add cancellation: allow the user to cancel the current tool call (send JSON-RPC cancel or specific method if supported by SDK).
- Acceptance criteria:
  - From `inspect`, user can pick a tool, fill inputs, execute, see results.
  - Cancellation works and is clearly indicated.
  - Optional `--json` returns machine‑readable output.

### 2) Resources

- Current:
  - `inspect` supports `listResources` and `readResource`.
- Gaps:
  - No subscription to resource list changes.
  - No exploration of resource `mimeType` and potential range/offset reading options if provided by servers.
- Plan:
  - In `inspect`, add an option to subscribe to resource updates (if supported by server). Display a console notification when resources change and offer to refresh.
  - Display resource metadata more prominently (name, uri, mimeType, description).
  - Add optional `--save <path>` to persist resource content to disk.
- Acceptance criteria:
  - Resource updates are surfaced during an `inspect` session when supported by server.
  - Users can save a resource to the local filesystem from `inspect`.

### 3) Prompts

- Current:
  - `inspect` supports `listPrompts` and `getPrompt(name, arguments)` with argument prompting.
- Gaps:
  - No subscription to prompt updates.
- Plan:
  - In `inspect`, add subscribe/unsubscribe to prompt updates and refresh UI on changes.
- Acceptance criteria:
  - Prompt list changes are surfaced during an `inspect` session.

### 4) Discovery (updates/subscriptions)

- Goal: Respond to server‑initiated updates for tools/resources/prompts (“discovery”) so users can see live changes without restarting.
- Plan:
  - In `inspect`, offer a “Live updates” toggle that subscribes to updates for all supported primitives.
  - Register notification handlers on the `Client` for list‑changed events and display concise toasts/logs.
  - Provide a “Refresh lists now” action to re‑fetch after change notifications.
- Acceptance criteria:
  - When servers add/remove tools/resources/prompts, `inspect` surfaces those changes in real time.

### 5) Roots

- Goal: Surface `roots` and enable navigation of resource roots.
- Plan:
  - In `inspect`, add a new “Roots” section:
    - Call `client.listRoots()` (or the SDK equivalent) and display the available roots.
    - Allow selecting a root to filter `resources` scoped beneath that root (when server supports it).
    - Show root metadata (name, uri/prefix, description).
- Acceptance criteria:
  - Roots can be listed and used to scope resource browsing in `inspect`.

### 6) Sampling

- Context: Some servers delegate language model generation to the client over MCP “sampling.” Most LLM apps implement this, but the CLI can optionally provide a simple backend so `inspect` can demonstrate and test server sampling flows.
- Plan:
  - Create a pluggable sampling backend module, e.g. `src/lib/sampling.ts`, that supports providers via env or flags:
    - Providers: `openai`, `anthropic`, `vertex`, `gemini`, or `mock`.
    - Configure credentials via env (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc.) and CLI flags (e.g., `--model <id>`).
  - In `inspect`, construct the `Client` with `capabilities.sampling` enabled when a provider is configured.
  - Register the sampling handler to call the selected provider and return model output back to the server.
  - Add an `--enable-sampling` flag to `inspect` to force-enable (fails with helpful guidance if no provider configured).
- Acceptance criteria:
  - A server that requests sampling from the client can complete its flows while running under `inspect` with sampling enabled.
  - Users can choose model/provider via flags or env.

### 7) Elicitation

- Context: Servers can request the client to elicit user input (e.g., secrets, confirmations, or structured fields) mid‑flow.
- Plan:
  - In `inspect`, construct the `Client` with `capabilities.elicitation` enabled.
  - Register an elicitation handler that:
    - Renders a clear prompt to the user using `inquirer`.
    - Supports structured elicitation (numbers, enums, booleans, masked inputs for secrets).
    - Returns the user’s inputs to the server.
  - Provide non‑interactive mode: if `--non-interactive` is passed, fail gracefully with guidance or allow answers via `--answers <json>`.
- Acceptance criteria:
  - Servers can request elicited inputs; `inspect` displays prompts and returns values, including masked secrets.

## Runner improvements (stdio and Streamable HTTP)

These components primarily pass JSON‑RPC between the LLM and the server. They are largely capability-agnostic and already suitable for “full coverage.” Minor enhancements can improve UX and resilience:

- Cancellation: map Ctrl+C during an active exchange to a JSON‑RPC cancellation message for the in‑flight id.
- Progress: pretty-print `progress` notifications to stderr while leaving JSON‑RPC on stdout untouched.
- Discovery: log list‑changed notifications (without altering pass‑through behavior).
- Session management (Streamable HTTP): already handles heartbeat, idle, retries, and graceful termination; no changes required beyond keeping SDK up to date.

Files:
- `src/commands/run/stdio-runner.ts`: add optional cancellation mapping and progress logging.
- `src/commands/run/streamable-http-runner.ts`: already supports heartbeat/idle/retry; add optional progress logging if desired.

## Interactive inspector changes (file-by-file)

- `src/commands/inspect.ts`
  - Build `Client` with declared client capabilities:
    - Always: logging, discovery/list change notifications.
    - Optional: sampling (if provider configured), elicitation (interactive only).
  - Register notification handlers: logging (already), tools/resources/prompts list changes, progress.
  - Expand the main loop to include:
    - Tools: execute and display results; support cancellation; `--json` output.
    - Resources: subscribe, browse metadata, save to disk.
    - Prompts: subscribe and refresh on change.
    - Roots: list and filter resources.
    - Settings: toggle live updates; configure sampling provider/model; switch non‑interactive mode.

- `src/utils/runtime.ts`
  - Add helpers to load sampling provider credentials from env and validate them.
  - Add helpers for masking secrets in logs.

- `src/lib/sampling.ts` (new)
  - Provider‑agnostic interface: `createSampler({ provider, model, ...creds })`.
  - Implement at least `mock` (no network) and one real provider (OpenAI or Anthropic) behind feature flags.

- `src/index.ts`
  - Flags for `inspect`:
    - `--enable-sampling` and `--model <id>`
    - `--non-interactive` and `--answers <json>`
    - `--json` for machine‑readable output

## Dependency and SDK alignment

- Keep `@modelcontextprotocol/sdk` current (already ^1.10.x). Periodically update to pick up new capability types and helpers.
- If adding real sampling providers:
  - Add optional deps (e.g., `openai`, `anthropic`) and guard with clear errors if not installed.
- Maintain Node 18+ baseline.

## UX and safety considerations

- Default to non‑destructive, read‑only operations unless the user explicitly confirms.
- Clearly separate stdout (strict JSON for LLMs) from stderr (human logs) in runners.
- Mask secrets in logs; provide a `--debug` mode that is still careful about sensitive values.
- Warn on remote servers (`utils/runtime.isRemote`) as already implemented.

## Testing strategy

- Unit tests for:
  - JSON Schema prompting (nested, arrays, enums, defaults, required).
  - Sampling provider selection and error paths.
  - Elicitation flows with `inquirer`.
  - Notification handlers (list changes, logging, progress).
- Integration smoke tests:
  - `inspect` against a known server exercising each capability (resources, tools, prompts, roots).
  - Sampling: a server that requests client sampling; verify successful round‑trip with the mock provider.
- Non‑interactive CI paths using canned `--answers <json>`.

## Example usage (after implementing changes)

- Inspect with live updates and sampling via OpenAI:
  - `npx @smithery/cli inspect <server-id> --enable-sampling --model gpt-4o --json`
- Save a resource to disk:
  - `npx @smithery/cli inspect <server-id>` → pick resource → choose “Save to file” → provide path
- Elicitation (interactive secrets):
  - Server triggers an elicitation request; CLI prompts for masked input and returns it.

## Acceptance checklist

- Tools: list, execute, progress display, cancel, JSON output.
- Resources: list, read, subscribe, save to disk.
- Prompts: list, get, subscribe to updates.
- Discovery: live updates toggle; notifications wired.
- Roots: list and scope resources.
- Sampling: optional provider support; mock provider included; works in `inspect`.
- Elicitation: interactive prompts implemented; non‑interactive fallback documented.
- Runners: remain pass‑through and LLM‑friendly; add optional cancellation mapping and progress stderr logs.

## Suggested implementation order

1) Tool execution + JSON Schema prompting (fastest user value)
2) Roots + resource improvements + save‑to‑disk
3) Discovery subscriptions (tools/resources/prompts)
4) Elicitation (interactive)
5) Sampling (mock → one real provider)
6) Cancellation mapping + progress logs in runners
7) Tests and docs polish

---

With these changes, the CLI will not only remain a great “install/run” tool for MCP servers, but also become a comprehensive, interactive reference client that exercises the full MCP capability surface end‑to‑end.