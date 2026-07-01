## Why

T3 Code supports multiple coding-agent providers (Codex, Claude, Cursor, Grok, OpenCode) but not Oh My Pi (omp). omp ships a standards-compliant ACP server (`omp acp` — JSON-RPC over stdio) with no private extensions, making it the simplest ACP provider to integrate. The provider driver architecture already has a proven ACP blueprint (Grok) that this change follows.

## What Changes

- Add a new `omp` provider driver kind to the contracts package (`ProviderDriverKind.make("omp")`), with default model, display name, and slug alias mappings.
- Add `OmpSettings` schema (binary path + optional home path) to contracts, plus its patch schema and `ServerSettings.providers.omp` entry.
- Implement `OmpDriver` (driver SPI), `OmpAcpSupport` (ACP spawn config + model resolution), `OmpAdapter` (ACP events → `ProviderRuntimeEvent` stream), and `OmpProvider` (snapshot/status checking) in the server.
- Implement `OmpTextGeneration` for git commit messages, PR content, branch names, and thread titles.
- Register `OmpDriver` in `BUILT_IN_DRIVERS` and add `OmpDriverEnv` to the `BuiltInDriversEnv` union.
- Add omp to the web UI's `PROVIDER_OPTIONS` list and `PROVIDER_ICON_BY_PROVIDER` icon mapping (2 lines total — the model picker, mode selector, and thinking toggle are driver-agnostic and already work).
- Add `docs/providers/omp.md` user documentation.

## Capabilities

### New Capabilities

- `omp-provider`: The Oh My Pi provider driver — ACP session lifecycle (new/load/resume/fork/close), prompt execution, permission handling, model selection across 80+ models, mode switching (default/plan), and thinking-level config. Includes text generation for git operations.

### Modified Capabilities

_None. No existing spec-level behavior changes; omp is additive._

## Impact

- **`packages/contracts`**: New `OmpSettings` schema, `ProviderDriverKind("omp")` registration in `model.ts` defaults/maps, `OmpSettingsPatch`, `ServerSettings.providers.omp` field.
- **`apps/server/src/provider`**: 7 new files (driver, ACP support, adapter, provider snapshot layer, service registration) + 2 edited files (`builtInDrivers.ts`, `builtInProviderCatalog.ts` if needed). Follows the Grok provider blueprint minus the `XAiAcpExtension` wrapper (omp is standards-only ACP).
- **`apps/server/src/textGeneration`**: 1 new file (`OmpTextGeneration.ts`).
- **`apps/web/src`**: 2 lines across `session-logic.ts` (`PROVIDER_OPTIONS`) and `providerIconUtils.ts` (`PROVIDER_ICON_BY_PROVIDER`) + an omp icon component.
- **`docs/providers`**: 1 new file (`omp.md`).
- **Dependencies**: None new. omp must be installed on the host (`omp` binary on `PATH`). Authentication is local-only via `~/.omp` credentials — no API keys, OAuth flows, or sensitive env vars to manage.
- **No breaking changes**: omp is a purely additive provider. Existing providers are untouched.
