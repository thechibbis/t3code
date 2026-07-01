## Context

T3 Code supports five provider drivers (Codex, Claude, Cursor, Grok, OpenCode) via a `ProviderDriver` SPI. Two of these — Grok and Cursor — connect to their agent over ACP (Agent Client Protocol), JSON-RPC over stdio. The ACP infrastructure is already factored into reusable pieces:

- `packages/effect-acp` — the ACP client (JSON-RPC framing, schema-validated requests/notifications).
- `apps/server/src/provider/acp/AcpSessionRuntime.ts` — session lifecycle (initialize → new/load/resume → prompt → session updates), generic over any ACP server.
- `apps/server/src/provider/acp/AcpRuntimeModel.ts` — parsing of ACP `configOptions`, `modes`, and `session/update` events into `ProviderRuntimeEvent`s. Driver-agnostic.
- `apps/server/src/provider/acp/AcpAdapterSupport.ts` — shared error-mapping (`mapAcpToAdapterError`) and permission-outcome translation.

Oh My Pi (`omp`) exposes `omp acp` — a standards-compliant ACP server. A live probe confirms: protocolVersion 1, `authMethods: [{ id: "agent" }]` (local credentials under `~/.omp`), `agentCapabilities: { loadSession, promptCapabilities: { embeddedContext, image }, sessionCapabilities: { list, fork, resume, close } }`. The `session/new` response returns `configOptions` for `model` (80+ models), `mode` (default/plan), and `thinking` (off/auto/high/xhigh), plus `modes` with `availableModes`.

Unlike Grok (which needs `XAiAcpExtension.ts` — a 432-line wrapper handling xAI's private `prompt/complete` notification and custom `askUserQuestion` request), omp sends only standard ACP notifications. No extension layer is needed.

The web UI is driver-agnostic by design: `ModelPickerContent` renders any model list via a virtualized searchable combobox; `TraitsPicker` renders config-option descriptors dynamically by id (`thinking`, `effort`, `fastMode`); mode changes flow through `current_mode_update` → `ModeChanged` events. Two UI touchpoints remain hardcoded: `PROVIDER_OPTIONS` (sidebar list) and `PROVIDER_ICON_BY_PROVIDER` (icon glyph map).

## Goals / Non-Goals

**Goals:**

- Add omp as a fully functional first-class provider: sessions, prompts, permissions, model selection, mode switching, thinking config.
- Reuse the existing ACP runtime/adapter infrastructure without duplicating Grok's extension layer.
- Support omp's full ACP capability set: session new/load/resume/fork/close, embedded context, image prompts.
- Wire text generation (commit messages, PR content, branch names, thread titles) through omp.
- Keep the change purely additive — no breaking changes to existing providers.

**Non-Goals:**

- omp plugin/marketplace support (the `session/update` `available_commands_update` advertises omp plugins, but those are omp-internal and not surfaced in T3 Code).
- Surfacing omp's slash commands (`/model`, `/compact`, `/share`, etc.) in T3 Code's UI — those are omp session-internal commands, not ACP protocol surface.
- Custom auth flows — omp uses local `~/.omp` credentials only; no API key or OAuth wiring.
- Refactoring the shared ACP adapter into a base class. Grok and Cursor adapters have enough divergence (error mapping, session context shape, provider-specific quirks) that a premature base class would be speculative. This change copies the Grok adapter pattern and strips the xAI extension.

## Decisions

### D1: Follow the Grok provider blueprint, minus the extension layer

**Decision**: Copy the Grok provider file structure (`Driver`, `AcpSupport`, `Adapter`, `Provider`, `Services`, `TextGeneration`) and remove the `XAiAcpExtension` wrapper.

**Rationale**: Grok is the closest structural match — same ACP transport, same session lifecycle, same text-generation pattern. The only structural difference is that omp is standards-only ACP (no private notifications), so the extension layer is unnecessary.

**Alternative considered**: Extract a shared `BaseAcpAdapter` from Grok/Cursor and have omp inherit it. Rejected because Grok's adapter has xAI-specific logic woven throughout (prompt-completion fallback, `askUserQuestion` handling, stop-reason normalization) and Cursor's has cursor-specific probing. A base class extracted now would either be too leaky or require a second extraction pass. Better to copy-and-trim, then extract a base class as a follow-up if a third standards-only ACP provider arrives.

### D2: Spawn command is `omp acp` with no arguments

**Decision**: `buildOmpAcpSpawnInput` returns `{ command: "omp", args: ["acp"], cwd, env }`.

**Rationale**: The `omp acp` command runs the ACP server over stdio with no subcommands or flags. Confirmed by `omp acp --help` ("Run Oh My Pi as an ACP server over stdio") and the live probe. The binary path is configurable via `OmpSettings.binaryPath` (default `"omp"`), same as Grok's `binaryPath`.

### D3: Single auth method — always `"agent"`

**Decision**: `resolveOmpAuthMethodId` always returns `"agent"` (omp's only auth method).

**Rationale**: omp's `initialize` response advertises exactly one auth method: `{ id: "agent", name: "Use existing local credentials", description: "...configured under ~/.omp" }`. No env-var-based auth, no API key. Unlike Grok (which branches between `xai.api_key` and `cached_token` based on `XAI_API_KEY`), omp has no auth branching. The resolver is a constant function kept for structural symmetry with the `AcpSessionRuntimeOptions.authMethodId` contract.

### D4: `OmpSettings` schema — binaryPath only, no homePath

**Decision**: `OmpSettings` has `enabled` + `binaryPath` (default `"omp"`) + `customModels` (hidden, defaults to `[]`).

**Rationale**: omp auth is local via `~/.omp` and is not configurable via environment variable or home-path override (unlike Codex's `CODEX_HOME` or Claude's `HOME` isolation). There is no `OMP_HOME` equivalent. A `homePath` field would be dead config. The `customModels` field follows Grok's pattern — hidden from the settings form, used internally if the driver needs to override the model list.

**Alternative considered**: Add a `homePath` for future-proofing. Rejected — adding dead config violates the project's "no unnecessary abstractions" principle. If omp adds home-path support later, it's a trivial schema extension.

### D5: Model resolution — dynamic from ACP, no curated defaults

**Decision**: `resolveOmpAcpBaseModelId` returns the provided model or `"umans/umans-glm-5.2"` (omp's reported `currentValue` in the `session/new` configOptions). The actual model list is discovered dynamically from the ACP `session/new` response at runtime.

**Rationale**: omp exposes 80+ models across `opencode-zen/*` and `umans/*` providers. Curating a static list would go stale immediately. The ACP runtime already extracts `currentModelId` from the session setup response via `AcpRuntimeModel.extractModelConfigId()`, and the model picker UI handles arbitrary-length lists via virtualization. The `DEFAULT_MODEL_BY_PROVIDER["omp"]` entry in contracts provides the fallback when no model is selected.

### D6: No CLI probe needed for auth

**Decision**: `OmpProvider.checkOmpProviderStatus` checks binary existence only (via `resolveSpawnCommand`), not auth state.

**Rationale**: Grok needs `GrokAcpCliProbe` to resolve auth state (API key vs cached token). omp has a single auth method and no auth state to probe — either `omp` is installed and `~/.omp` has credentials, or it isn't. The provider snapshot will show "available" if the binary exists, and the first `session/new` will surface any auth failure as an `AcpError`.

### D7: Maintenance capabilities — manual-only, same as Grok

**Decision**: Use `makeManualOnlyProviderMaintenanceCapabilities` + `makeStaticProviderMaintenanceResolver`, same as Grok.

**Rationale**: omp updates are managed by the omp installer, not by T3 Code. No update-check endpoint, no version comparison. The maintenance resolver is a no-op that reports "manual" for both update checking and installation.

## Risks / Trade-offs

**[Adapter code duplication]** → The `OmpAdapter` will share ~70% of its structure with `GrokAdapter` (session context, permission handling, prompt settlement, event stream wiring). Acceptable for now — the shared logic is interleaved with provider-specific types and error paths. A follow-up extraction into a `BaseAcpAdapter` is viable once omp proves the pattern is stable. Risk: low. Mitigation: keep `OmpAdapter` focused; if a third standards-only ACP provider appears, extract then.

**[omp binary not installed]** → If `omp` is not on `PATH`, the provider snapshot shows "unavailable" and the driver won't auto-bootstrap. This is the same behavior as all other providers. Risk: none beyond existing pattern.

**[80+ models in picker]** → The model picker uses a virtualized combobox with fuzzy search, so 80+ entries render fine. Risk: UX discoverability — users may not find the model they want. Mitigation: favorites and jump-key shortcuts already exist; the search indexes `driverKind`, `providerDisplayName`, `name`, and `shortName`.

**[omp session/update with `available_commands_update`]** → omp sends a `session/update` with `available_commands_update` containing 40+ slash commands. This is a standard ACP `sessionUpdate` type, but T3 Code's `AcpRuntimeModel.parseSessionUpdateEvent` may not have a case for it. If unhandled, it will be silently ignored (the parser has a default fallthrough). Risk: low — no functional impact, just unused data. Mitigation: verify during implementation that `available_commands_update` doesn't cause a parse warning.

**[No extension layer assumption]** → This design assumes omp sends only standard ACP notifications based on the `initialize` + `session/new` probe. A full `prompt` cycle spike is needed during implementation to confirm no private notifications appear during prompt execution. Risk: if omp does send private notifications, an `OmpAcpExtension.ts` will be needed (adding ~200-400 lines). Mitigation: the adapter structure allows adding an extension wrapper later without restructuring, same as how Grok wraps the base runtime with `makeXAiPromptCompletionRuntime`.
